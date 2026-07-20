# HabitMate — Bot specification

**Archetype:** custom

**Voice:** encouraging and concise — write every user-facing message, button label, error, and empty state in this voice.

HabitMate is a private habit-tracking Telegram bot that helps users build and maintain habits through gentle reminders, one-tap logging, and progress tracking. It supports flexible schedules, private habit history, and milestone celebrations without being cloying. Users can track current and longest streaks, completion percentages, and view weekly summaries.

> This is the complete contract for the bot. Implement EVERY entry point, flow, feature, integration, and edge case below. The completeness review checks the bot against this document after each build pass.

## Primary audience

- Individuals who want a lightweight, private habit tracker inside Telegram
- Users who prefer gentle, non-punitive nudges and concise progress views

## Success criteria

- Users can create and track habits with one-tap logging
- Users receive timely, timezone-aware reminders
- Streaks and progress are accurately tracked and displayed
- Users can view weekly summaries and milestone achievements

## Entry points

Every feature must be reachable from the bot's command/button surface (button-first; only /start and /help are slash commands).

- **/start** (command, actor: user, command: /start) — Open the main menu
- **+ New Habit** (button, actor: user, callback: habit:new) — Create a new habit
- **/dashboard** (command, actor: user, command: /dashboard) — View the dashboard with active habits and progress
- **My Habits** (button, actor: user, callback: dashboard:view) — View the dashboard with active habits and progress

## Flows

### Onboarding
_Trigger:_ /start

1. Greet user and explain purpose
2. Ask for timezone (quick picks + detect option)
3. Offer default timezone if skipped
4. Quick tutorial: create first habit or view sample habits

_Data touched:_ User

### Create Habit
_Trigger:_ habit:new

1. Prompt for title (required)
2. Select schedule type (daily / selected weekdays / N times per week)
3. Set preferred local reminder time
4. Optional description
5. Start date (default today)
6. Optional tags
7. Initial pause? (no by default)

_Data touched:_ Habit

### N-times-per-week Schedule
_Trigger:_ habit:schedule:weekly

1. Ask how many times per week (1–7)
2. Optionally select preferred weekdays or let bot pick evenly

_Data touched:_ Habit

### Daily Reminder
_Trigger:_ reminder:send

1. Send gentle reminder at chosen local time
2. Show inline buttons: Done, Skip today, Mark failed, Snooze (10 min), More (opens habit details)

_Data touched:_ Occurrence

### One-tap Logging
_Trigger:_ button:tap

1. Record one occurrence for that habit
2. Update streaks immediately
3. Update reminder message to reflect action
4. Disable further Done taps for that habit for that date

_Data touched:_ Occurrence, Streaks & stats

### Manual Edits
_Trigger:_ habit:edit

1. Show habit detail view with quick controls
2. Allow Mark today Done / Skip / Fail
3. Edit habit (schedule/time/title)
4. Pause/Resume
5. Delete
6. Change past days via date picker

_Data touched:_ Habit, Occurrence, Streaks & stats

### Weekly Summary
_Trigger:_ /dashboard

1. Show compact list of all active habits
2. Display title, small emoji/status, current streak, best streak, percent this week
3. Show quick 'Today' action button
4. Separate weekly view with 7-day grid per habit

_Data touched:_ Streaks & stats

### Milestone Notifications
_Trigger:_ milestone:trigger

1. Detect milestone (7-day streak, 30-day streak, 50% completion month)
2. Show non-intrusive milestone note with encouraging copy
3. Allow user to turn milestone notes off in settings

_Data touched:_ Milestone history

## Data entities

Durable data (must survive a restart) uses the toolkit's persistent store, never in-memory maps.

- **User** _(retention: persistent)_ — Telegram account with user-specific settings
  - fields: Telegram id, timezone, language, settings, notification tokens/state
- **Habit** _(retention: persistent)_ — User-defined habit with schedule and metadata
  - fields: title, description, schedule type, preferred local reminder time, state (active/paused), creation date, tags
- **Occurrence** _(retention: persistent)_ — Daily entry for a habit with action metadata
  - fields: date (user-local), status (done/skipped/failed/unmarked), timestamp of action, source (reminder tap/manual), immutable id
- **Streaks & stats** _(retention: persistent)_ — Progress tracking for habits
  - fields: current streak length, best streak length, completion percentage, milestone flags
- **Settings** _(retention: persistent)_ — User-specific preferences
  - fields: notification mute windows, preferred reminder behavior, language, privacy flag
- **Milestone history** _(retention: persistent)_ — Record of milestones shown to avoid repeats
  - fields: milestone type, date shown

## Integrations

- **Telegram** (required) — Bot API messaging
Call external APIs against their real contract (correct endpoints, ids, params); credentials from env. Do not fake responses.

## Owner controls

- Configure default timezone fallback
- Set default behavior for streak breaks (skip vs fail)
- Enable/disable milestone notifications
- Set default language (currently RU)
- Configure snooze duration (10 min default)

## Notifications

- Daily reminders with inline buttons
- Milestone notifications with encouraging messages
- Streak updates after logging
- Dashboard updates with progress changes

## Permissions & privacy

- All habit data is private and stored per-user
- No data is shared by default
- User can set privacy flag to 'private only'

## Edge cases

- User changes timezone after habit creation
- User edits past occurrences
- User skips or fails a habit with custom streak rules
- User creates a habit with N times per week without specifying weekdays
- User tries to mark a habit done multiple times in the same day

## Required tests

- End-to-end test of habit creation and logging flow
- Test timezone handling and reminder timing
- Verify streak calculations with different schedule types
- Test milestone notifications and their suppression
- Ensure private data remains isolated between users

## Assumptions

- Users will be asked for timezone at onboarding; if skipped, bot uses Telegram-reported timezone or UTC as fallback
- Reminders send once at the chosen local time with an optional single snooze (10 minutes)
- N-times-per-week schedule without explicit weekdays will be treated as 'any days' with the user responsible for logging
- Single-tap logging enforces one recorded occurrence per habit per local date
- Skipped vs Failed: 'Skip' records an intentional skip (does not break streak depending on user preference — default: skip breaks streak), 'Fail' records a broken day — default behaviour: skips break streaks to keep expectations strict
- All habit data is private and stored per-user; no sharing features are implemented unless the owner requests them later
- Language default is Russian; user can change language in settings
