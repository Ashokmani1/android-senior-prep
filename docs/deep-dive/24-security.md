# Security

Security is the topic where "it compiles and runs" is worth nothing. A senior is expected to reason about the **threat model** first (what is the attacker, what do they want, what can they touch) and then reach for the platform primitive that matches — not sprinkle Base64 "encryption" and root checks and call it hardened. This module covers the build-time hardening (R8), the transport layer (NSC + pinning, which lives in full detail in [M11 Networking](11-networking-advanced.md)), the hardware root of trust (Keystore), user-presence binding (BiometricPrompt), encryption at rest (Jetpack Security, detailed in [M23 Storage](23-storage.md)), server-side attestation (Play Integrity), release signing, and the everyday app-surface mistakes that get apps popped.

!!! abstract "What a senior is expected to own here"
    - Push secrets into **hardware** (Keystore/StrongBox) so a rooted device or a stolen backup still can't extract the key material.
    - Bind a decryption key to **user presence** with a `CryptoObject`, and know Class 2 vs Class 3 and why it matters.
    - Treat the client as **hostile**: no secret in code survives decompilation; trust decisions belong on the server (Play Integrity), not in a client-side `isRooted()`.
    - Lock down the **app surface** — exported components, intent redirection, tapjacking — because that's where real CVEs live, not in your crypto.

---

## R8 / ProGuard

R8 is the default Android shrinker/optimizer/obfuscator (it replaced ProGuard; it still consumes ProGuard-syntax rules). It does four jobs in one pass on release builds: **shrinking** (tree-shakes unreachable classes/members), **optimization** (inlining, class merging, dead-branch removal), **obfuscation** (renames to `a`, `b`, `c` — shrinks the DEX and raises the reverse-engineering cost), and **resource shrinking** (with `shrinkResources true`).

```kotlin
android {
    buildTypes {
        release {
            isMinifyEnabled = true      // R8 code shrink + obfuscate + optimize
            isShrinkResources = true    // drop unused resources
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

R8's static analysis can't see reflection, JNI, or serialization entry points, so you must **keep** them:

```proguard
# Retrofit models hit by Gson reflection
-keep class com.example.api.model.** { *; }
# Enums used by valueOf()/values()
-keepclassmembers enum * { public static **[] values(); public static ** valueOf(java.lang.String); }
```

!!! danger "The obfuscation-vs-crash-reporting trap"
    Obfuscated release stack traces are unreadable (`a.b.c(SourceFile:1)`). R8 emits `mapping.txt` per build — **archive it** (CI artifact + upload to Crashlytics/Play). Without the exact `mapping.txt` for that `versionCode`, a production crash is undebuggable. Deobfuscate with `retrace mapping.txt obfuscated_trace.txt`. Losing the mapping file is a one-way door.

Security note: obfuscation is **defense in depth, not a security control**. It slows a human reader; it does not stop a determined attacker with `jadx`. Never rely on it to hide a secret.

!!! note "Deeper coverage"
    The full R8 treatment — optimization passes, `keep` rule authoring, DEX/size impact, baseline profiles — lives in [M33 Performance](33-performance.md). Here we only care that it obfuscates and that you must preserve `mapping.txt`.

---

## Network Security Config & certificate pinning

Transport trust should be **declarative**, in `res/xml/network_security_config.xml`, not a hand-rolled `X509TrustManager` (a code-based trust override is an instant review reject). Since API 28 cleartext is off by default; NSC is where you relax it (debug only) or tighten it.

```xml
<network-security-config>
    <domain-config cleartextTrafficPermitted="false">
        <domain includeSubdomains="true">api.example.com</domain>
        <pin-set expiration="2026-12-31">
            <pin digest="SHA-256">AAAA…currentLeafOrIntermediateSPKI=</pin>
            <pin digest="SHA-256">BBBB…BACKUP_pin_you_already_own=</pin>
        </pin-set>
    </domain-config>
</network-security-config>
```

Certificate/public-key **pinning** hard-codes which server SPKI hashes you trust, defeating a user-installed or compromised CA doing MITM. The senior-level danger is **rotation**: a shipped pin that expires or a server cert rotated without a pre-published backup pin **bricks every installed app** — the client can't be patched fast enough. Always ship a backup pin, set `expiration`, and rotate ahead of the server. Full strategy and the pin-rotation war story are in [M11 Networking → TLS/SSL pinning](11-networking-advanced.md#tlsssl-pinning-strategies).

---

## Android Keystore system

The Keystore is a **hardware-backed** key container. Key material is generated inside and **never leaves** the secure hardware — the TEE (Trusted Execution Environment) or, on supporting devices, the **StrongBox** dedicated secure element (a separate tamper-resistant chip, API 28+). Your app gets an opaque handle; crypto operations are executed *inside* the secure world. Even a fully rooted OS or a lifted disk image cannot export the raw key.

Keys are constrained at generation time via `KeyGenParameterSpec` — the constraints (purposes, block modes, auth requirements) are **enforced by hardware**, not by your code, so they hold even if the app process is compromised.

```kotlin
val spec = KeyGenParameterSpec.Builder(
    "user_data_key",
    KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
)
    .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
    .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
    .setKeySize(256)
    .setUnlockedDeviceRequired(true)          // usable only while screen unlocked
    .setIsStrongBoxBacked(true)               // demand the secure element (see fallback below)
    .setUserAuthenticationRequired(true)      // ties key use to biometric/credential
    .setUserAuthenticationParameters(
        30,                                    // valid 30s after auth (time-based)…
        KeyProperties.AUTH_BIOMETRIC_STRONG    // …or 0 = per-use, requires a CryptoObject
    )
    .build()

val kpg = KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore")
kpg.init(spec)               // throws StrongBoxUnavailableException if no secure element
val secretKey = kpg.generateKey()
```

!!! warning "StrongBox availability & auth invalidation"
    - `setIsStrongBoxBacked(true)` throws `StrongBoxUnavailableException` on devices without a secure element — catch it and retry the spec with StrongBox off (TEE-backed). Never fail hard.
    - With `setUserAuthenticationRequired(true)`, enrolling a **new fingerprint / changing the lock screen invalidates the key** (`KeyPermanentlyInvalidatedException`) by design. Catch it, delete the key, re-provision, and re-prompt — don't crash-loop.
    - Use `setUnlockedDeviceRequired` + attestation (`setAttestationChallenge`) when a server needs to *prove* a key lives in real hardware.

The same key can be created for signing (`PURPOSE_SIGN`/`PURPOSE_VERIFY`, e.g. an EC key for request signing) instead of encryption — the purpose flags are the hardware-enforced boundary.

---

## Biometric authentication

`BiometricPrompt` (androidx.biometric) renders the **system** biometric sheet — you never touch raw fingerprint/face data; the OS returns only a success/failure. Two integration levels:

- **Just gate the UI:** `authenticate(promptInfo)` — the callback tells you "user is present." Good enough for "unlock this screen," but a rooted attacker who hooks your callback can fake success.
- **Cryptographically bind it:** `authenticate(promptInfo, CryptoObject(cipher))`. The `CryptoObject` wraps a `Cipher`/`Signature`/`Mac` initialized with a Keystore key created with `setUserAuthenticationRequired(true)` and **per-use** auth. The key is only *unlocked for that one operation* by the successful biometric. There is no callback to hook — without a real auth the cipher physically won't decrypt. This is the senior-correct pattern for protecting real secrets.

**Class 2 (Weak) vs Class 3 (Strong):** Class 3 sensors meet a strict spoof-acceptance bar and are the **only** class allowed to gate a `CryptoObject` (unlock a Keystore key). Class 2 can authenticate a session but cannot release crypto keys. Request the floor you need with `setAllowedAuthenticators(BIOMETRIC_STRONG)`.

**Device-credential fallback:** add `DEVICE_CREDENTIAL` so users without/failing biometrics can use PIN/pattern/password. Note you **cannot** combine `DEVICE_CREDENTIAL` with `BIOMETRIC_WEAK` and a `CryptoObject` on older APIs — validate the combination with `BiometricManager.canAuthenticate(authenticators)` before prompting.

```kotlin
val promptInfo = BiometricPrompt.PromptInfo.Builder()
    .setTitle("Unlock secure data")
    .setAllowedAuthenticators(BIOMETRIC_STRONG or DEVICE_CREDENTIAL)
    .build()

val cipher = getCipherInitializedWithKeystoreKey(Cipher.DECRYPT_MODE, iv)
val prompt = BiometricPrompt(activity, ContextCompat.getMainExecutor(activity),
    object : BiometricPrompt.AuthenticationCallback() {
        override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
            // Only NOW is the key unlocked — use the returned cipher, not a fresh one.
            val plaintext = result.cryptoObject!!.cipher!!.doFinal(ciphertext)
        }
        override fun onAuthenticationError(code: Int, msg: CharSequence) { /* lockout / cancel */ }
    })

prompt.authenticate(promptInfo, BiometricPrompt.CryptoObject(cipher))
```

### Flow: BiometricPrompt + Keystore CryptoObject

```mermaid
sequenceDiagram
    participant App
    participant KS as Keystore (TEE/StrongBox)
    participant BP as BiometricPrompt (System UI)
    participant HW as Biometric Sensor (Class 3)

    App->>KS: getKey("user_data_key") [auth-required, per-use]
    App->>App: Cipher.init(DECRYPT_MODE, key, iv)
    Note over App: Cipher is created but key is LOCKED
    App->>BP: authenticate(promptInfo, CryptoObject(cipher))
    BP->>HW: capture & match (in secure world)
    HW-->>KS: auth token (HAT) proves user presence
    KS-->>BP: unlock key for THIS operation only
    BP-->>App: onAuthenticationSucceeded(result.cryptoObject)
    App->>App: result.cryptoObject.cipher.doFinal(ciphertext) ✓
    Note over App,KS: No valid auth ⇒ doFinal throws; nothing to bypass
```

---

## Encryption at rest

Use Jetpack Security (`androidx.security:security-crypto`) rather than rolling AES yourself. A `MasterKey` lives in the Keystore (hardware-backed) and wraps the data-encryption keys; the library handles IVs, AEAD, and key wrapping correctly.

```kotlin
val masterKey = MasterKey.Builder(context)
    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
    // .setUserAuthenticationRequired(true, 15)  // optional: gate master key on auth
    .build()

val prefs = EncryptedSharedPreferences.create(
    context, "secret_prefs", masterKey,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,   // keys deterministic
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM  // values AEAD
)
prefs.edit().putString("auth_token", token).apply()   // encrypted on disk

// Whole-file encryption (media, DB exports, downloaded docs):
val encFile = EncryptedFile.Builder(
    context, File(filesDir, "report.pdf"), masterKey,
    EncryptedFile.FileEncryptionScheme.AES256_GCM_HKDF_4KB
).build()
encFile.openFileOutput().use { it.write(bytes) }
```

- **`EncryptedSharedPreferences`** — for small key/value secrets (tokens, flags). Keys and values are both encrypted; keys use SIV so lookups stay deterministic.
- **`EncryptedFile`** — for arbitrary files, chunked AEAD streaming.
- Room isn't covered by Jetpack Security — use **SQLCipher** for an encrypted DB.

!!! note
    The Jetpack Security library was **deprecated** in recent AndroidX releases; the primitives and the pattern (MasterKey in Keystore wrapping data keys) remain the interview-correct mental model, and many production apps still ship it. Full storage-layer treatment and alternatives in [M23 Storage](23-storage.md).

### Keystore vs EncryptedSharedPreferences

| | Android Keystore | EncryptedSharedPreferences |
|---|---|---|
| **Stores** | Cryptographic **keys** | Arbitrary **key/value data** |
| **Key location** | Never leaves secure hardware | MasterKey in Keystore; data on disk |
| **Best for** | The root key, signing keys, auth-bound keys | Tokens, credentials, small secrets |
| **Auth binding** | `setUserAuthenticationRequired` + CryptoObject | Indirect (via MasterKey auth) |
| **Extractable?** | No (opaque handle) | Ciphertext on disk; useless without MasterKey |
| **Wrong use** | Storing bulk data | Storing a raw key you should keep in Keystore |

---

## Play Integrity API

Play Integrity **replaced SafetyNet Attestation** (fully deprecated). It answers one question a client can never answer for itself: *is this a genuine app, on a genuine device, from a genuine user?* The verdict is requested on-device but the token is **verified server-side** (decrypt/validate against Google) — a verdict read only on the client is trivially spoofed and worthless.

Three verdict groups:

- **Device integrity** — `MEET_DEVICE_INTEGRITY` (genuine Play-certified device), plus stronger `MEET_STRONG_INTEGRITY` (hardware-backed, recent security update) and weaker `MEET_BASIC_INTEGRITY`. Absence flags emulators, rooted, or tampered OS.
- **App integrity** — is the running binary the unmodified one you signed/uploaded (`PLAY_RECOGNIZED`), not a repackaged/patched APK.
- **Account/licensing** — did this user actually acquire the app from Play.

!!! warning "Server-side or it's theatre"
    Always verify the integrity token on your backend and gate the *server response*, not a client `if`. Enforcing integrity in client code is removed by the same attacker you're defending against. Budget for false negatives (custom ROMs, non-GMS devices) — degrade gracefully rather than hard-blocking legitimate users.

---

## App signing

Every APK/AAB is signed; Android verifies the signature chain on install and on update (an update must be signed by the **same** key). The scheme has evolved:

| Scheme | Since | Protects | Notes |
|---|---|---|---|
| **v1 (JAR)** | Always | Per-file (META-INF) | Slow verify; doesn't cover ZIP metadata; vulnerable to Janus (pre-v2). Keep only for API <24. |
| **v2** | API 24 (N) | Whole-APK block hash | Faster, tamper-evident over the entire archive. |
| **v3** | API 28 (P) | v2 + **key rotation** | Lets you rotate the signing key with a proof-of-continuity lineage. |
| **v3.1 / v4** | API 30–33 | v4 = streaming hash (ADB Incremental) | v4 sidecar signature enables fast incremental install; complements, doesn't replace, v2/v3. |

**Play App Signing** splits the keys, and this is the part interviewers probe:

- **Upload key** — what *you* sign the AAB with before uploading. If leaked, you reset it with Google; low blast radius.
- **App signing key** — Google holds it (HSM) and re-signs the artifact delivered to users. This is the identity users' devices trust for updates. If it leaked without Play App Signing you'd be unable to ship updates ever again — Play holding it is the safety net.

!!! danger "Losing the signing key = a dead app"
    Without Play App Signing, losing your keystore means you can never update the app — you'd have to publish a brand-new listing and lose all installs/reviews. Enroll in Play App Signing and back up the **upload** key in a secure secret store. In CI, inject the keystore from encrypted secrets; never commit `.jks` or its password.

---

## Common threats & mitigations

!!! danger "Secrets in code / BuildConfig — don't"
    `BuildConfig.API_KEY`, string constants, and `strings.xml` all ship in the APK. `apktool`/`jadx` extract them in seconds; obfuscation only renames the *holder*, not the value. **There is no such thing as a client-side secret.** Mitigations: keep real secrets server-side; use short-lived tokens minted by your backend (gated by Play Integrity); for keys that *must* live on-device, generate them in Keystore so they're never a literal. The NDK is marginally harder to read, not safer.

| Threat | What it is | Mitigation |
|---|---|---|
| **Exported components** | An `Activity`/`Service`/`Receiver`/`Provider` reachable by other apps | Set `android:exported="false"` unless truly public (mandatory to declare since API 31). Enforce a `signature`-level permission if it must be exported. Never trust incoming `Intent` extras. |
| **Intent redirection** | You receive an `Intent` and blindly `startActivity`/`grantUriPermission` on a nested Intent/URI from it | Validate component, action, and URI against an allowlist; never forward an attacker-supplied Intent or grant it URI permissions. |
| **Tapjacking** | A malicious overlay tricks the user into tapping your sensitive control | `android:filterTouchesWhenObscured="true"` (or `setFilterTouchesWhenObscured`) on consent/security buttons; drop touches when the window is obscured. |
| **Root detection** | Detecting rooted/tampered devices | Useful signal, **defeatable** (Magisk hides root). Treat as risk input feeding Play Integrity, not a hard client gate. Don't ship a false sense of security. |
| **Insecure storage** | Tokens in plain SharedPreferences / external storage | EncryptedSharedPreferences/EncryptedFile; never secrets on shared/external storage. |
| **PendingIntent** | Mutable PendingIntent lets a receiver rewrite it | `FLAG_IMMUTABLE` (required since API 31) unless you have a specific mutable need. |
| **WebView** | `addJavascriptInterface`, `file://` access, loading untrusted content | Disable JS interface unless needed; `setAllowFileAccess(false)`; load only trusted origins. |
| **Backup leakage** | Auto Backup exfiltrates secrets to cloud | Exclude secrets via `dataExtractionRules`/`fullBackupContent`; don't back up token stores. |

---

## Interview Q&A

**Q1. Why is a key in the Android Keystore safer than an AES key you store in `EncryptedSharedPreferences`?**
Keystore key material is generated inside secure hardware (TEE, or StrongBox secure element) and **never leaves** it — your app only ever holds an opaque handle, and crypto runs *inside* the secure world. Even root or a disk image can't export it. `EncryptedSharedPreferences` stores *ciphertext* on disk; it's safe only because its MasterKey lives in the Keystore. So Keystore protects the *root of trust*; EncryptedSharedPreferences protects *bulk data* under a key that ultimately depends on the Keystore.
*Follow-up: what's StrongBox and when would you demand it?* A dedicated tamper-resistant chip separate from the main SoC (API 28+). Demand it (`setIsStrongBoxBacked(true)`) for high-value keys, but catch `StrongBoxUnavailableException` and fall back to TEE.

**Q2. Walk me through binding a decryption key to biometrics so a hooked callback can't bypass it.**
Generate a Keystore key with `setUserAuthenticationRequired(true)` and **per-use** validity (auth window 0). Init a `Cipher` with it — the key is locked. Pass `CryptoObject(cipher)` to `BiometricPrompt.authenticate`. Only a successful **Class 3** auth unlocks the key for that single operation, and you must use the cipher returned in `result.cryptoObject`. Since decryption physically fails without a real auth, there's no boolean callback to hook.
*Follow-up: why must it be Class 3, not Class 2?* Only Class 3 (Strong) meets the spoof-resistance bar required to release Keystore keys; Class 2 can gate a UI session but can't unlock a CryptoObject-bound key.

**Q3. A teammate wants to protect an API key by moving it from `strings.xml` into `BuildConfig` and enabling R8 obfuscation. Good enough?**
No. Both ship inside the APK; `jadx`/`apktool` recover them trivially, and R8 only renames the *symbol*, never the literal value. There is no client-side secret. Move it server-side, mint short-lived tokens from the backend (gated by Play Integrity), and for on-device keys generate them in the Keystore so no literal exists.
*Follow-up: is the NDK safer?* Marginally harder to read, not fundamentally safer — still extractable. Not a security boundary.

**Q4. Why did Google split app signing into an upload key and an app signing key, and what breaks if you lose each?**
With Play App Signing, Google holds the **app signing key** (in an HSM) and re-signs the delivered artifact — that's the identity devices trust for updates. You sign with an **upload key** Google can reset if leaked. Losing the *upload* key → reset with Google, minor disruption. Without Play App Signing, losing your *signing* key means you can never update the app (updates must match the original signature) — you'd lose the entire listing. So the split limits blast radius and makes key loss recoverable.
*Follow-up: what does v3 add over v2?* Key **rotation** with a proof-of-continuity lineage, so you can change the signing key without invalidating installs.

**Q5. How do you actually enforce "block rooted/emulated devices" for a banking feature?**
Not with a client `isRooted()` — root hiders defeat it and the check runs in the attacker's process. Use **Play Integrity**: request device/app/account verdicts on-device, but **verify the token server-side** and gate the sensitive *server response*, not a client branch. Combine device-integrity signals with app-integrity (unmodified binary) and degrade gracefully for legitimate non-GMS/custom-ROM users rather than hard-blocking.
*Follow-up: what replaced SafetyNet here, and why is server verification non-negotiable?* Play Integrity replaced SafetyNet Attestation. A verdict read only on the client is spoofable by the very attacker you're defending against; the cryptographic token must be validated on your backend to mean anything.
