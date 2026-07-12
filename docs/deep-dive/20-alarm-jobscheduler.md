# AlarmManager & JobScheduler

Two low-level scheduling primitives that predate WorkManager. A senior Android engineer must know *why* they still exist, *when* one is correct, and *why you almost never call `JobScheduler` directly anymore*. The one-line mental model:

> **Exact wall-clock time → `AlarmManager`. Deferrable, constraint-driven, guaranteed work → `WorkManager`. `JobScheduler` is the OS engine WorkManager sits on — you rarely touch it yourself.**

---

## AlarmManager

`AlarmManager` fires a `PendingIntent` (or, on API 21+, an `OnAlarmListener`) at a point in time. It is the only API that can wake the device to run at a **precise wall-clock moment**. It does *not* run your code — it delivers a broadcast/service/activity trigger; your work happens in the receiver, which is subject to the ~10s `BroadcastReceiver` limit (offload to `goAsync()` or a `WorkManager` job).

### Clock types

Two orthogonal choices: **which clock** (wall-clock vs boot-elapsed) and **whether it wakes the device**.

| Type | Clock base | Wakes device? | Survives reboot? | Use for |
|------|-----------|---------------|------------------|---------|
| `RTC` | `System.currentTimeMillis()` (wall clock) | No — fires when device next wakes | No (RTC baseline changes) | Non-urgent wall-clock events; UI refresh while awake |
| `RTC_WAKEUP` | Wall clock | **Yes** | No | Alarm clock, calendar reminder at a specific date/time |
| `ELAPSED_REALTIME` | `SystemClock.elapsedRealtime()` (since boot, incl. sleep) | No | Timer resets on reboot | "N minutes from now", relative delays that shouldn't wake device |
| `ELAPSED_REALTIME_WAKEUP` | Elapsed since boot | **Yes** | Timer resets on reboot | Relative wake-up ("wake me in 30 min") independent of wall-clock/timezone changes |

!!! tip "Wall-clock vs elapsed — the senior distinction"
    Use `RTC*` when the trigger is tied to a **calendar time** ("7:00 AM") — but beware timezone/clock changes and NTP corrections shifting it. Use `ELAPSED_REALTIME*` when the trigger is a **relative interval** ("90 minutes of cook time"); it is immune to the user changing the clock or crossing timezones. Alarms do **not** persist across reboot — re-register in a `BOOT_COMPLETED` receiver.

### Scheduling methods

| Method | Precision | Doze-safe? | Notes |
|--------|-----------|------------|-------|
| `set()` | **Inexact** (OS batches; window widens) | No | API 19+ is inexact by design — OS coalesces to save battery |
| `setWindow(type, start, length, pi)` | Inexact within a window you define | No | You express tolerance; OS picks the moment. Battery-friendly |
| `setExact()` | Exact | **No** — deferred in Doze | Requires exact-alarm permission on 12+. Delayed until maintenance window in Doze |
| `setExactAndAllowWhileIdle()` | Exact | **Yes** — one shot pierces Doze | Rate-limited to ~1 per 9 min per app. The correct choice for a real alarm |
| `setAndAllowWhileIdle()` | Inexact | Yes (fires in Doze) | Inexact + Doze-piercing; same ~9 min throttle |
| `setRepeating()` | **Inexact** on API 19+ | No | For exact repetition, reschedule a one-shot each time it fires |
| `setInexactRepeating()` | Inexact, pre-set intervals | No | OS batches wakeups across apps |
| `setAlarmClock(AlarmClockInfo, pi)` | **Exact**, highest priority | **Yes** — always fires on time | Surfaces the next-alarm status-bar icon; exempt from Doze deferral; **no exact-alarm permission needed**. Use only for user-visible alarm-clock UX |

!!! warning "`setRepeating` is a trap"
    Since API 19, `setRepeating()` is **inexact** — intervals are batched and drift. If you need exact periodic firing, schedule a single `setExactAndAllowWhileIdle()` and re-schedule the next one from inside the receiver. But if it's *deferrable* periodic work, that's a `PeriodicWorkRequest` in WorkManager, not AlarmManager.

### Doze & App Standby

Doze (API 23+) is the reason exact alarms are hard. When the screen is off, unplugged, and stationary, the device enters Doze and **defers** normal alarms, network, jobs, and wakelocks to periodic **maintenance windows** (which grow less frequent over time).

- `set()` / `setExact()` / `setRepeating()` → **deferred** until the next maintenance window in Doze.
- `setExactAndAllowWhileIdle()` / `setAndAllowWhileIdle()` → allowed to fire in Doze, **but throttled** to roughly once every 9 minutes per app.
- `setAlarmClock()` → exempt; fires on time (and briefly brings the device out of Doze before it fires).

### Exact-alarm permission (Android 12+/13+)

Google clamped down on exact alarms because they wake the device and drain battery. From **Android 12 (API 31)**:

- `SCHEDULE_EXACT_ALARM` — a **special app access**. Declared in the manifest; on 31–32 it is *granted by default* but the user (or OS) can revoke it, and it is visible in Settings → "Alarms & reminders."
- `USE_EXACT_ALARM` (**Android 13 / API 33+**) — a **normal permission**, granted at install, no user toggle. **BUT** Google Play restricts it to apps whose *core function* is an alarm clock or calendar. Using it elsewhere risks policy rejection.

The correct runtime pattern:

1. Declare the permission in the manifest.
2. Before scheduling, call `AlarmManager.canScheduleExactAlarms()` (API 31+).
3. If `false`, either fall back to an inexact alarm **or** send the user to `ACTION_REQUEST_SCHEDULE_EXACT_ALARM` settings — never crash.
4. Listen for `ACTION_SCHEDULE_EXACT_ALARM_PERMISSION_STATE_CHANGED` to re-schedule when granted.

!!! danger "Play policy: exact alarms must be justified"
    Only declare `USE_EXACT_ALARM` if the app is genuinely an alarm/clock/calendar app. Everyone else uses `SCHEDULE_EXACT_ALARM` and must **gracefully degrade** when it's denied. Calling `setExact*` without permission on 31+ throws `SecurityException`.

### When exact alarms are legitimate

- **Alarm clock / wake-up app** — the user expects it to fire at exactly 6:30 AM.
- **Calendar / event reminders** — "Meeting in 10 minutes" tied to a specific instant.
- **Medication / dosing reminders** — time-critical to the user.

Everything else — sync, backup, upload, cache cleanup, "refresh feed every few hours," prefetch — is **deferrable** and belongs in WorkManager. If a reviewer asks "why an exact alarm?" and your answer isn't "the user set a specific time," it's the wrong tool.

### Kotlin: exact-and-allow-while-idle with permission check

```kotlin
class ReminderScheduler(private val context: Context) {

    private val alarmManager = context.getSystemService(AlarmManager::class.java)

    /** @return true if scheduled exactly; false if we degraded to inexact. */
    fun scheduleReminder(triggerAtMillis: Long, requestCode: Int): Boolean {
        val pendingIntent = PendingIntent.getBroadcast(
            context,
            requestCode,
            Intent(context, ReminderReceiver::class.java),
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE,
        )

        // API 31+: exact alarms require SCHEDULE_EXACT_ALARM to be granted.
        val canExact = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
            alarmManager.canScheduleExactAlarms()
        } else {
            true // pre-12: no special access needed
        }

        return if (canExact) {
            // Pierces Doze, fires at the wall-clock instant. Best for real reminders.
            alarmManager.setExactAndAllowWhileIdle(
                AlarmManager.RTC_WAKEUP,
                triggerAtMillis,
                pendingIntent,
            )
            true
        } else {
            // Degrade gracefully — never crash, never spam the user with dialogs.
            alarmManager.setAndAllowWhileIdle(
                AlarmManager.RTC_WAKEUP,
                triggerAtMillis,
                pendingIntent,
            )
            false
        }
    }

    /** Send the user to grant exact-alarm access, only if the UX truly needs it. */
    fun requestExactAlarmPermission() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
            context.startActivity(
                Intent(Settings.ACTION_REQUEST_SCHEDULE_EXACT_ALARM).apply {
                    data = Uri.fromParts("package", context.packageName, null)
                    addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
                },
            )
        }
    }
}
```

!!! note "Receiver hygiene"
    The `PendingIntent` must be `FLAG_IMMUTABLE` (mandatory on API 31+). Keep `ReminderReceiver.onReceive()` under ~10s; for anything heavier, acquire a short wakelock via `goAsync()` or enqueue an expedited `OneTimeWorkRequest`. Alarms die on reboot — re-register from a `RECEIVE_BOOT_COMPLETED` receiver.

---

## JobScheduler

`JobScheduler` (API 21+) is the platform service that runs **deferrable background work under constraints** — network, charging, idle, storage-not-low. It batches jobs across apps to preserve battery and is Doze-aware. It is the foundation WorkManager builds on for API 23+.

### JobInfo — the constraint spec

You describe *what conditions* the job needs; the OS decides *when* to run it.

```kotlin
val job = JobInfo.Builder(JOB_ID, ComponentName(context, SyncJobService::class.java))
    .setRequiredNetworkType(JobInfo.NETWORK_TYPE_UNMETERED) // Wi-Fi only
    .setRequiresCharging(true)
    .setRequiresDeviceIdle(false)
    .setPersisted(true)          // survive reboot — needs RECEIVE_BOOT_COMPLETED
    .setBackoffCriteria(30_000L, JobInfo.BACKOFF_POLICY_EXPONENTIAL)
    .setPeriodic(15 * 60_000L)   // min period is 15 min, enforced by OS
    .build()

val scheduler = context.getSystemService(JobScheduler::class.java)
scheduler.schedule(job)
```

Common constraints: `setRequiredNetworkType`, `setRequiresCharging`, `setRequiresDeviceIdle`, `setRequiresStorageNotLow`, `setMinimumLatency` (one-shot only), `setOverrideDeadline`, `setPersisted`, `setBackoffCriteria`, `setPeriodic`.

### JobService — the execution contract

```kotlin
class SyncJobService : JobService() {

    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)

    // Runs on the MAIN thread. You MUST move real work off it yourself.
    override fun onStartJob(params: JobParameters): Boolean {
        scope.launch {
            try {
                doSync()                       // heavy work off the main thread
                jobFinished(params, /* wantsReschedule = */ false)
            } catch (e: Exception) {
                jobFinished(params, /* wantsReschedule = */ true) // apply backoff
            }
        }
        return true // TRUE = work continues on a background thread; keep me alive
    }

    // Called when constraints are no longer met (e.g. Wi-Fi dropped) or timeout.
    override fun onStopJob(params: JobParameters): Boolean {
        scope.coroutineContext.cancelChildren()
        return true // TRUE = reschedule with backoff; FALSE = drop it
    }
}
```

The three contract traps seniors are expected to nail:

1. **`onStartJob` runs on the main thread.** The framework does *not* give you a background thread — you must dispatch work yourself (coroutine, executor, thread). Blocking here causes ANRs.
2. **Return value of `onStartJob`**: `true` means "I've handed work to another thread, keep the job alive — I'll call `jobFinished()` when done." `false` means "work is already complete, release me now."
3. **`jobFinished(params, wantsReschedule)`** must be called explicitly when async work ends, or the job is considered running forever (and eventually killed). `wantsReschedule = true` re-queues with your backoff policy. `onStopJob` returning `true` reschedules; return the value that matches whether the work is worth retrying.

`JobParameters` carries `jobId`, the `PersistableBundle`/`Bundle` extras, the triggered content URIs (for content-observer jobs), and `stopReason` (API 31+ tells you *why* you were stopped — timeout, constraint lost, app standby, user action).

!!! warning "Why you don't write this by hand anymore"
    Raw `JobService` gives you: no built-in threading, manual reboot persistence, no unified chaining, min-SDK 21, and no observable state. **WorkManager wraps all of it** — picks `JobScheduler` on 23+, handles threading via `CoroutineWorker`/`ListenableWorker`, persists across reboot for free, and adds chaining, `LiveData`/`Flow` observation, and expedited work. In 2024+ code, calling `JobScheduler` directly is a smell unless you have a very specific reason (e.g. `TRIGGER_CONTENT_URI` jobs, or you're implementing the library layer itself).

---

## Decision matrix: WorkManager vs JobScheduler vs AlarmManager

| Dimension | WorkManager | JobScheduler (raw) | AlarmManager |
|-----------|-------------|--------------------|--------------|
| **Best for** | Guaranteed, deferrable background work | (Legacy) constraint-based deferrable work | Exact wall-clock time-of-day triggers |
| **Timing model** | "Eventually, when constraints met" | "Eventually, when constraints met" | "At *this* instant" |
| **Constraints (network/charging/idle)** | ✅ Rich | ✅ Rich | ❌ None |
| **Survives reboot** | ✅ Automatic | ⚠️ `setPersisted(true)` only | ❌ Re-register on `BOOT_COMPLETED` |
| **Survives app kill / force-stop** | ✅ (until force-stop) | ✅ | Force-stop cancels alarms |
| **Doze behavior** | Deferred to maintenance; expedited option | Deferred to maintenance windows | Only `*AllowWhileIdle` / `setAlarmClock` pierce Doze |
| **Threading provided** | ✅ Worker gives you a thread | ❌ You dispatch off main yourself | ❌ Receiver runs briefly on main |
| **Chaining / observability** | ✅ Chains, `Flow`/`LiveData` | ❌ | ❌ |
| **Min API** | 14 (Jetpack) | 21 | 1 |
| **Wakes the device precisely** | ❌ | ❌ | ✅ (`*_WAKEUP`) |

```mermaid
flowchart TD
    A[Need to schedule work] --> B{Must run at an<br/>exact wall-clock time<br/>the USER chose?}
    B -->|Yes| C{Is the app core function<br/>an alarm/clock/calendar?}
    C -->|Yes| D[AlarmManager<br/>setAlarmClock or<br/>setExactAndAllowWhileIdle<br/>+ USE_EXACT_ALARM]
    C -->|No| E[AlarmManager<br/>setExactAndAllowWhileIdle<br/>+ SCHEDULE_EXACT_ALARM<br/>degrade if denied]
    B -->|No| F{Is completion<br/>guaranteed / deferrable<br/>with constraints?}
    F -->|Yes| G{Runs while app<br/>is foreground only?}
    G -->|No| H[WorkManager<br/>OneTime / Periodic<br/>WorkRequest]
    G -->|Yes| I[Coroutine / lifecycle scope<br/>no scheduler needed]
    F -->|"Time-critical but<br/>brief & user-initiated"| J[WorkManager<br/>expedited work<br/>setExpedited]
    H -.->|On API 23+ WorkManager<br/>uses JobScheduler internally| K[JobScheduler<br/>do NOT call directly]
    style K stroke-dasharray: 5 5
```

**The rules of thumb, ranked:**

1. **Deferrable + must eventually complete** (sync, upload, backup, cleanup, prefetch) → **WorkManager**. This is the default answer for ~90% of background scheduling.
2. **Exact time-of-day the user set** (alarm, reminder, calendar) → **AlarmManager** with `setExactAndAllowWhileIdle` / `setAlarmClock`, gated on exact-alarm permission with graceful degradation.
3. **Time-critical but short and user-triggered** → WorkManager **expedited** work (foreground service quota) rather than an alarm.
4. **JobScheduler directly** → almost never in app code. Use it only for niche cases (content-URI trigger jobs) or when you can't take a Jetpack dependency. Otherwise let WorkManager own it.

---

## Interview Q&A

**Q1. Difference between `RTC_WAKEUP` and `ELAPSED_REALTIME_WAKEUP`, and when would using the wrong one bite you?**
`RTC_WAKEUP` triggers relative to wall-clock time (`System.currentTimeMillis()`); `ELAPSED_REALTIME_WAKEUP` triggers relative to time since boot (`SystemClock.elapsedRealtime()`, which includes deep sleep). Both wake the device. Use `RTC_WAKEUP` for calendar/alarm-clock events tied to a real date-time. Use `ELAPSED_REALTIME_WAKEUP` for relative intervals ("in 30 minutes"). Getting it wrong bites you when the user changes the timezone or the system clock: an `RTC` alarm shifts (a "7 AM" alarm can fire early/late or twice), while `ELAPSED_REALTIME` is immune. Conversely, using `ELAPSED_REALTIME` for a fixed calendar time drifts because the boot baseline has no notion of wall-clock date.
*Follow-up: why don't RTC alarms survive a reboot?* Because pending alarms live in RAM in the AlarmManagerService and are cleared on reboot; you must re-register them from a `BOOT_COMPLETED` receiver (and mark jobs `setPersisted` for JobScheduler).

**Q2. What does Doze do to `setExact()`, and how do you actually get an alarm to fire on time?**
In Doze, `setExact()` (and `set`, `setRepeating`) are deferred until the next maintenance window, which grows increasingly infrequent — so "exact" becomes "sometime later." To fire in Doze you need `setExactAndAllowWhileIdle()` (throttled to ~once per 9 min per app) or, for genuine alarm-clock UX, `setAlarmClock()`, which is exempt from Doze deferral and always fires on time (it also shows the status-bar alarm icon). If work is deferrable, the right answer is *don't fight Doze* — use WorkManager.
*Follow-up: why the 9-minute throttle on `setExactAndAllowWhileIdle`?* To stop apps from using it as a general wakeup loop that defeats Doze's battery savings; the OS rate-limits Doze-piercing alarms per app.

**Q3. Walk me through the exact-alarm permission changes in Android 12 and 13.**
Android 12 (API 31) introduced `SCHEDULE_EXACT_ALARM` as a special app access: declared in the manifest, grantable/revocable by the user in Settings ("Alarms & reminders"), and required before `setExact*` calls (else `SecurityException`). On 31–32 it's granted by default but can be revoked. Android 13 (API 33) added `USE_EXACT_ALARM`, a normal install-time permission with no user toggle — but Google Play restricts it to apps whose core purpose is alarms/clock/calendar. So: general apps use `SCHEDULE_EXACT_ALARM`, check `canScheduleExactAlarms()`, and degrade gracefully; only true alarm apps use `USE_EXACT_ALARM`.
*Follow-up: user revokes `SCHEDULE_EXACT_ALARM` while your alarm is pending — what happens?* Existing exact alarms are cancelled by the system, and you receive `ACTION_SCHEDULE_EXACT_ALARM_PERMISSION_STATE_CHANGED`; you should re-evaluate and either reschedule (if regranted) or fall back to inexact.

**Q4. In a `JobService`, what thread does `onStartJob` run on, and what does its return value mean?**
`onStartJob` runs on the app's **main thread**. The framework does not hand you a worker thread, so you must offload heavy work yourself (coroutine/executor). The return value tells the framework whether work is ongoing: `true` = "I've dispatched work to another thread, keep me alive; I'll call `jobFinished()` when done," `false` = "work already finished synchronously, release me." Forgetting to call `jobFinished()` after returning `true` leaves the job "running" until the system kills it, and you lose your backoff/reschedule intent.
*Follow-up: what's `onStopJob` for and what should it return?* It's called when constraints are lost or the job times out (`stopReason` on API 31+ says why). Cancel your in-flight work and return `true` to reschedule with your backoff policy, or `false` to abandon it.

**Q5. When would you reach for AlarmManager over WorkManager, and why is calling JobScheduler directly discouraged?**
Reach for AlarmManager only when work must run at a precise wall-clock instant the user chose — alarm clock, calendar reminder, medication reminder — because it's the only API that wakes the device at an exact time. Everything deferrable (sync, upload, backup, cleanup, periodic refresh) goes to WorkManager, which guarantees eventual execution under constraints and survives reboot. Calling `JobScheduler` directly is discouraged because WorkManager already wraps it on API 23+, adding threading, reboot persistence, chaining, observability, expedited work, and a single API across SDK levels — hand-rolling `JobService` reintroduces bugs (main-thread work, manual persistence) the library already solved.
*Follow-up: is there any case where you'd still touch JobScheduler directly?* Niche ones: content-URI trigger jobs (`setTriggerContentUri`) that WorkManager doesn't expose as cleanly, or environments where you can't add the Jetpack dependency. Otherwise, no.
