# Requirements Document

## Introduction

This feature addresses three targeted bug fixes and one feature addition in the ORVEX NET BOT Telegram bot (TypeScript/Node.js with Telegraf). The changes are:

1. **Announcement broadcast rewrite** — The current `ANNOUNCEMENT_SCENE` iterates over users sequentially with a 50 ms sleep and calls `ctx.telegram.sendMessage` directly, bypassing the `NotificationService` batch logic and its 403-handling. It must be replaced with a call to `NotificationService`.
2. **Image support in announcements** — The announcement wizard must ask for an optional photo/GIF URL after the admin enters the text and before the confirmation step.
3. **`adminMiddleware` silent failure** — Non-admin users who trigger a callback query are left with a spinner (no `answerCbQuery` call), freezing the Telegram client. The middleware must answer the callback query before stopping.
4. **Duplicate `admin_view_stats` button** — The inline keyboard in `/admin` lists the button `📈 Ver estadísticas` twice (rows 11 and 13). The duplicate must be removed.
5. **Unimplemented `admin_notify` action** — The `/admin` panel keyboard has a `📢 Enviar Notificación Global` button with callback data `admin_notify`, but no handler exists in `setupAdminRoutes`. The action must be implemented.

## Glossary

- **Bot**: The Telegraf-based Telegram bot running in `src/index.ts`.
- **AdminController**: The module `src/controllers/adminController.ts` that registers admin commands and callback actions.
- **adminMiddleware**: The Telegraf middleware function exported from `AdminController` that gates admin-only handlers.
- **AnnouncementScene**: The Telegraf `WizardScene` with ID `ANNOUNCEMENT_SCENE` defined in `src/scenes/announcement.ts`.
- **NotificationService**: The class in `src/services/notifications.ts` that provides rate-limited, batch broadcast with 403 auto-block and optional channel forwarding.
- **broadcastAnnouncement**: A new method to be added to `NotificationService` that accepts a text and an optional media URL and delegates to the existing `broadcastToAll` method.
- **CallbackQuery**: A Telegram update sent when a user taps an inline keyboard button; must always be answered with `answerCbQuery` to dismiss the loading spinner.
- **Supabase**: The PostgreSQL-as-a-service backend accessed via the `supabase` client.
- **WizardScene**: A multi-step Telegraf scene that advances through numbered handler steps.

---

## Requirements

### Requirement 1 — Announcement broadcast via NotificationService

**User Story:** As an admin, I want announcements sent through the optimized `NotificationService` so that messages are delivered in rate-limited batches, blocked users are auto-marked, and the official channel also receives the message.

#### Acceptance Criteria

1. WHEN the admin confirms an announcement in `ANNOUNCEMENT_SCENE`, THE AnnouncementScene SHALL call `NotificationService.broadcastAnnouncement(text, photoUrl?)` instead of iterating over users directly.
2. THE NotificationService SHALL expose a `broadcastAnnouncement(text: string, photoUrl?: string)` method that delegates to the existing `broadcastToAll(text, markup?, photoUrl?)` method with no inline keyboard markup.
3. WHEN `broadcastToAll` receives a 403 error for a user, THE NotificationService SHALL update the `bloqueado` column of that user's row in the `usuarios` table to `true`.
4. WHEN `broadcastAnnouncement` completes, THE NotificationService SHALL return an object containing `successCount` and `failCount` integer values.
5. WHEN the announcement broadcast finishes, THE AnnouncementScene SHALL reply to the admin with the final `successCount` and `failCount` values.

### Requirement 2 — Optional image support in announcement wizard

**User Story:** As an admin, I want to attach an optional photo or GIF to a global announcement so that the broadcast message is visually richer.

#### Acceptance Criteria

1. WHEN the admin submits the announcement text in step 2 of `ANNOUNCEMENT_SCENE`, THE AnnouncementScene SHALL prompt the admin to send a photo/GIF URL or tap a "Skip" button before advancing to the confirmation step.
2. WHEN the admin provides a non-empty URL string in the image step, THE AnnouncementScene SHALL store the URL in `ctx.wizard.state.photoUrl` and advance to the confirmation step.
3. WHEN the admin taps the "Skip" button in the image step, THE AnnouncementScene SHALL set `ctx.wizard.state.photoUrl` to `undefined` and advance to the confirmation step.
4. WHEN the admin confirms the announcement, THE AnnouncementScene SHALL pass `ctx.wizard.state.photoUrl` as the second argument to `broadcastAnnouncement`.
5. WHEN the admin provides a URL in the image step, THE AnnouncementScene confirmation message SHALL include a note indicating that a media attachment will be sent.

### Requirement 3 — adminMiddleware must answer callback queries for non-admins

**User Story:** As a regular user who accidentally triggers an admin callback, I want the bot to respond immediately so that the Telegram loading spinner is dismissed and my client is not frozen.

#### Acceptance Criteria

1. WHEN a user who is not in the `administradores` table triggers a callback query that passes through `adminMiddleware`, THE adminMiddleware SHALL call `ctx.answerCbQuery()` before returning without calling `next()`.
2. WHEN a user who is not in the `administradores` table triggers a text command that passes through `adminMiddleware`, THE adminMiddleware SHALL return without calling `next()` and without sending any message.
3. IF `ctx.from` is absent, THEN THE adminMiddleware SHALL return without calling `next()` and without calling `ctx.answerCbQuery()`.

### Requirement 4 — Remove duplicate stats button from admin panel

**User Story:** As an admin, I want the admin panel keyboard to have no duplicate buttons so that the interface is clean and navigable.

#### Acceptance Criteria

1. THE AdminController SHALL render the `/admin` inline keyboard with exactly one button for `admin_view_stats`.
2. WHEN the `/admin` command is executed, THE Bot SHALL display the inline keyboard with no repeated callback data values across any two buttons.

### Requirement 5 — Implement admin_notify action

**User Story:** As an admin, I want the "Enviar Notificación Global" button in the admin panel to work so that I can broadcast a custom notification without going through the announcement scene.

#### Acceptance Criteria

1. WHEN an admin taps the `📢 Enviar Notificación Global` button (callback data `admin_notify`), THE AdminController SHALL answer the callback query and enter `ANNOUNCEMENT_SCENE`.
2. THE AdminController SHALL register a handler for the `admin_notify` callback data that is guarded by `adminMiddleware`.
3. WHEN `ANNOUNCEMENT_SCENE` is entered via `admin_notify`, THE AnnouncementScene SHALL behave identically to when it is entered via `admin_send_announcement`.
