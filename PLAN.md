# GibbonChat – Module Plan

A Gibbon module that lets a teacher chat with a class (course class) or form group.

## Background (from earlier research)

- I found no official or community Gibbon module for class or course chat. I checked gibbonedu.org/extend, the GibbonEdu GitHub org, yookoala/GibbonEduModules and GitHub searches.
- Built-in **Messenger** sends one-way announcements to a class or form group by email, SMS or the wall. It is not a conversation.
- **Planner** lesson comments are limited and are not a chat.
- Gibbon is not a VLE, so forums and chat are left to modules.
- `module-stream` and `module-feed` (official) may be worth reading for UI and patterns. I have not opened either yet.
- Starting point: the official template `GibbonEdu/module-gibbonStarterModule`.

## Goals

1. Each course class and form group has a chat room that its teachers and students can use.
2. Access comes from Gibbon's existing enrolment, so the module keeps no membership list of its own.
3. Teachers can moderate: delete messages, mute a student, lock the room.
4. Optional email or notification for new messages, using Gibbon's `NotificationSender` and its mailer.
5. Arabic, German and English, with RTL support.

## Scope

### MVP (v1.0)
- One room per `gibbonCourseClassID` and per `gibbonFormGroupID`. A room is created the first time someone opens it.
- A page that lists "My Chats" (the rooms the current user belongs to).
- A room view with the message history (paginated, newest at the bottom) and a text input.
- Plain-text messages with safe link handling.
- New messages arrive by AJAX polling every 5–10 seconds.
- Teachers can delete any message. Students can delete their own within N minutes.
- The room is scoped to the current school year.

### v1.1
- File and image attachments using Gibbon's upload handling.
- Unread counts and an "@mention" notification.
- Teacher controls: lock the room (read-only for students), mute a student.
- Room setting: parents may join read-only. Off by default.

### Later
- WebSockets or SSE instead of polling, if hosting allows.
- Ad-hoc group rooms (an activity or a custom group).
- A dashboard hook or widget showing recent unread messages.
- Export a room transcript as PDF.

## Data model

| Table | Key columns |
|---|---|
| `gibbonChatRoom` | `gibbonChatRoomID`, `type` ENUM('Class','FormGroup'), `gibbonCourseClassID` NULL, `gibbonFormGroupID` NULL, `gibbonSchoolYearID`, `status` ENUM('Open','Locked'), `allowParents` Y/N, `timestampCreated` |
| `gibbonChatMessage` | `gibbonChatMessageID`, `gibbonChatRoomID`, `gibbonPersonID`, `body` TEXT, `attachment` NULL, `deleted` Y/N, `gibbonPersonIDDeleted` NULL, `timestamp` |
| `gibbonChatMember` | `gibbonChatRoomID`, `gibbonPersonID`, `lastReadMessageID`, `muted` Y/N. This holds per-user state only; it does not decide who may enter. |

Indexes: `(gibbonChatRoomID, gibbonChatMessageID)` on messages, and a unique index on `(type, gibbonCourseClassID, gibbonFormGroupID, gibbonSchoolYearID)`.

## Access rules

- **Class room:** `gibbonCourseClassPerson` rows with role `Teacher`, `Assistant` or `Student` and no `Left` status.
- **Form group room:** the tutors and EAs on `gibbonFormGroup`, plus the students enrolled through `gibbonStudentEnrolment` for that year.
- **Admin:** an "all rooms" action for oversight.
- Every endpoint re-checks membership on the server. The client is never trusted.

## Module structure

```
GibbonChat/
├── manifest.php            # name, version, tables, actions/permissions
├── CHANGEDB.php            # migrations per version
├── version.php
├── moduleFunctions.php
├── src/
│   ├── Domain/ChatRoomGateway.php
│   ├── Domain/ChatMessageGateway.php
│   ├── Domain/ChatMemberGateway.php
│   └── ChatAccess.php       # membership checks
├── chat_rooms.php          # My Chats
├── chat_room.php           # room view
├── chat_manage.php         # admin: all rooms
├── chat_settings.php       # module settings
├── ajax/
│   ├── messages_fetch.php  # ?roomID&afterID → JSON
│   ├── message_send.php
│   └── message_delete.php
├── js/chat.js
├── css/chat.css
└── i18n/ (ar_SA, de_DE)
```

## Permissions (manifest actions)

- **View My Chats:** Teacher, Student. Parent is optional.
- **Moderate Chats:** Teacher.
- **Manage All Chats:** Admin.
- **Chat Settings:** Admin.

## Settings

Polling interval, the window in which students can delete their own messages, max message length, attachments on/off, parents on/off, and email notifications on/off.

## Security

- CSRF tokens on every POST, using the Gibbon form token.
- Escape output, and run rich content through Gibbon's HTML sanitiser.
- Rate-limit message sending per user.
- Check the type and size of uploaded files.

## Implementation steps

1. Scaffold from `module-gibbonStarterModule`, then write the manifest and tables.
2. Gateways and the `ChatAccess` membership checks.
3. The My Chats page.
4. The room view, plus the fetch and send AJAX endpoints with polling JS.
5. Moderation (delete, lock).
6. Settings page.
7. i18n (ar and de), with an RTL CSS check.
8. Test on a live Gibbon instance (v27+) with teacher, student and admin accounts.

## Open questions

- Should parents see class chats at all? Read-only, or never?
- Should students be able to message each other privately, or only in the room?
- How long should messages be kept: per school year, or forever?
- Which Gibbon version is targeted?
