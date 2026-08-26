# TaskGrade — Task & Deadline Manager with Smart Reminders

A Flutter app with a clean dashboard for tasks + due times, and a local
notification that fires exactly 15 minutes before each task is due.

## Project structure

```
taskgrade/
├── pubspec.yaml
├── android/
│   └── AndroidManifest-additions.xml   # merge into the generated manifest
└── lib/
    ├── main.dart                        # app entry, theme, init
    ├── models/
    │   └── task_model.dart              # Task + Grade (priority) model
    ├── services/
    │   └── notification_service.dart    # core 15-min-before scheduling logic
    ├── providers/
    │   └── task_provider.dart           # state + persistence (SharedPreferences)
    ├── screens/
    │   └── dashboard_screen.dart        # main dashboard UI
    └── widgets/
        ├── add_task_dialog.dart         # task + time input fields
        └── task_tile.dart               # list item UI
```

## How the 15-minute reminder works

In `notification_service.dart`, `scheduleReminder(task)`:
1. Computes `task.dueAt.subtract(Duration(minutes: 15))`.
2. Converts it to a timezone-aware `TZDateTime` (so DST/timezone changes
   don't shift the alert).
3. Calls `zonedSchedule(...)` with `androidScheduleMode: exactAllowWhileIdle`,
   which tells Android to fire the alarm at the exact time even in Doze mode.
4. Builds the dynamic message: `"Only 15 minutes left for your task: ${task.title}"`.

Every add/edit/delete/complete action in `task_provider.dart` calls back into
this service, so notifications always stay in sync with the task list —
completing or deleting a task cancels its pending reminder automatically.

## Option A — Build a real APK with zero local install (GitHub Actions)

`.github/workflows/build.yml` is included and does everything for you: it
scaffolds the missing `android/` platform folder, patches in the required
notification permissions, and runs `flutter build apk --release` — all on
GitHub's servers.

1. Create a new **public or private** GitHub repo and push this whole
   `taskgrade/` folder to it (the `.github/workflows/build.yml` file must
   be at that exact path).
   ```bash
   cd taskgrade
   git init
   git add .
   git commit -m "Initial commit"
   git branch -M main
   git remote add origin https://github.com/<you>/taskgrade.git
   git push -u origin main
   ```
2. On GitHub, open the **Actions** tab of the repo — the "Build APK"
   workflow starts automatically on push (or click **Run workflow** to
   trigger it manually).
3. Wait for the run to go green (a few minutes), open it, and download the
   **taskgrade-apk** artifact from the bottom of the run page — it's a zip
   containing `app-release.apk`.
4. Transfer that `.apk` to an Android phone (email it to yourself, use
   Google Drive, or `adb install app-release.apk`) and install it — you'll
   need to allow "install from unknown sources" the first time.

No Flutter or Android Studio installation needed on your machine for this path.

## Option B — Build locally

1. **Install Flutter**: https://docs.flutter.dev/get-started/install
   Verify with `flutter doctor`.
2. From inside this `taskgrade/` folder, scaffold the platform files (only
   needed once):
   ```bash
   flutter create --org com.taskgrade --project-name taskgrade .
   ```
   This adds `android/`, `ios/`, etc. without touching your existing `lib/`.
3. **Merge the Android permissions** — open the newly generated
   `android/app/src/main/AndroidManifest.xml` and add the entries from
   `android/AndroidManifest-additions.xml` into it.
4. Install dependencies and either run on a connected device/emulator, or
   build the release APK directly:
   ```bash
   flutter pub get
   flutter run                    # to test live
   flutter build apk --release    # to produce build/app/outputs/flutter-apk/app-release.apk
   ```

**iOS note**: no manifest edits needed — `Info.plist` permission prompts are
handled by the `requestPermissions` call in `notification_service.dart`. iOS
builds require a Mac with Xcode either way (GitHub Actions can do this too
with a `macos-latest` runner, ask if you want that variant).

**Quick test of the notification**: add a task with a due time ~16 minutes
from now, background the app, and wait — the reminder should appear at the
15-minutes-before mark.

## What I'd extend first

1. **Real grade/GPA tracking** — right now "grade" is a priority tag (A/B/C).
   If you meant academic grades, add a `Course` model, a grades list per
   course, and a simple GPA calculator screen.
2. **Recurring tasks** — daily/weekly repeat option, using
   `matchDateTimeComponents` in `zonedSchedule`.
3. **Multiple reminder offsets** — let users pick 15 min / 1 hr / 1 day
   before, not just the hardcoded 15.
4. **Cloud sync** — swap `SharedPreferences` for Firebase/Supabase so tasks
   sync across devices; the `TaskProvider` interface is already isolated so
   this is a contained change.
5. **Calendar view** — a month/week grid view of tasks alongside the list.
6. **Snooze action** on the notification itself (flutter_local_notifications
   supports action buttons) to reschedule 5/10 minutes later.
