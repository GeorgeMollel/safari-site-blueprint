# Prompt: build an "Asili Portal"-style operations dashboard on another website

Copy everything below the line into a new Claude Code session that has access to
the target site (WordPress MCP / Hostinger API / SSH). Fill in the `{{…}}` values
first. Delete any module in section 6 you don't want.

---

Build a private operations portal for my business on **{{SITE_URL}}** (WordPress).
It must be a real plugin, not page-builder widgets, and match the spec below
exactly. Keep the source in a local folder `{{LOCAL_FOLDER}}` and deploy from there.

## 1. Business details

- Business name: **{{BUSINESS_NAME}}** — the portal is called "{{BUSINESS_NAME}} Portal"
- What we sell: {{e.g. safaris / tours / services}}; a lead is one enquiry for one trip/job
- Brand tint colour: light mode **{{TINT_LIGHT}}** (must pass 4.5:1 with white text), dark mode **{{TINT_DARK}}** (lighter step, black text on it)
- Logo: {{path to horizontal SVG lockup}} plus a white version for dark backgrounds
- Login background photo: {{path to own photo}}; faded workspace photo: {{path}}, ≤1400px, ≤100 KB
- Currency {{USD}}, time zone {{Africa/Dar_es_Salaam}}, reply target {{4}} hours, "stale lead" after {{5}} days
- Where enquiries come from today: {{form plugin / custom post type / email}} — read them, do not change the public form

## 2. Architecture

- Plugin folder `{{prefix}}-portal/`: `{{prefix}}-portal.php` (defines a `VERSION` constant, bumped on every deploy and appended as `?v=` to CSS/JS), `includes/class-*.php` (one class per module), `views/shell.php` (the only server-rendered page), `assets/portal.css`, `assets/portal.js`, `assets/icons/`.
- Front-end route at `/portal/` (rewrite rule + query var), sub-routes `/portal/{screen}/`. Not inside wp-admin.
- **Single-page app**: shell.php prints one JSON boot payload (leads, settings, user, permissions); portal.js renders every screen client-side with History API routing. No framework, no build step, vanilla JS, one CSS file.
- Add-on screens (costing, chat, notes) live in their own JS files that register with the app (`window.PORTAL_EXTS.push({views, titles, navBadge, todayCard, leadCard, action, prefix})`), so the main file stays manageable.
- REST namespace `{{prefix}}-portal/v1` for every write (sync, leads/{id}, settings, tasks, etc.), nonce + capability checked. Optimistic UI: update immediately, roll back with a toast on failure.
- All portal data lives in post meta prefixed `_{{p}}_` on the lead posts (stage, value, follow-up date, owner, activity log, first-reply time, last-touch time, lost reason, **lead source** and referrer name, trip start/end, guide, vehicle, tasks array, costings) and options prefixed `{{p}}_` for settings. Sample data is flagged `_{{p}}_demo=1` so it can be removed in one click.

## 3. Roles and access

- New role **"{{Planner}}"** with capability `{{p}}_portal`; planners are redirected out of wp-admin to /portal/.
- Administrators are shown as **"Director"** (label only, via a `wp_roles_init` filter) and see an extra "Director" nav group.
- Login page (wp-login.php) restyled: always dark, full-bleed brand photo, white logo instead of the WordPress logo, frosted-glass card, tint-coloured button.

## 4. Design system — Apple Human Interface Guidelines

Follow Apple's HIG strictly; it should feel like a native macOS/iOS app.

- **Font**: system stack only: `-apple-system, BlinkMacSystemFont, "SF Pro Text", "SF Pro", "Helvetica Neue", "Segoe UI", system-ui, sans-serif` (display: "SF Pro Display"; mono: `ui-monospace, "SF Mono", Menlo`). Apple type ramp (Large Title 34, Title 1 28, Title 2 22, Title 3 20, Headline 17 semibold, Body 17, Callout 16, Subhead 15, Footnote 13, Caption 12).
- **Every colour is a CSS token on `:root`**; dark mode is purely a token swap, under `@media (prefers-color-scheme: dark) { :root:not([data-theme="light"]) {…} }` and again under `:root[data-theme="dark"]`. Theme toggle: System / Light / Dark.
- Light tokens: `--bg #f2f2f7`, `--surface #fff`, `--surface-2 #f2f2f7`, `--surface-3 rgba(118,118,128,.12)`, `--fill-2 rgba(120,120,128,.16)`, `--line rgba(60,60,67,.18)`, `--line-2 rgba(60,60,67,.29)`, `--ink #000`, `--ink-2 rgba(60,60,67,.78)`, `--ink-3 rgba(60,60,67,.62)`, sidebar `--side rgba(246,246,248,.82)` (translucent + backdrop blur), `--bar rgba(249,249,249,.86)`, `--good #248a3d`, `--warn #c93400`, `--bad #d70015`, each with a `-soft` 13–15% tint.
- Dark tokens: `--bg #000` (true black), `--surface #1c1c1e`, `--surface-2 #2c2c2e`, `--line rgba(84,84,88,.6)`, `--ink #fff`, `--ink-2 rgba(235,235,245,.7)`, `--ink-3 rgba(235,235,245,.5)`, `--side rgba(30,30,32,.78)`, `--bar rgba(22,22,23,.86)`, `--good #30d158`, `--warn #ff9f0a`, `--bad #ff453a`.
- **One app tint only** (the brand colour), tokens `--tint`, `--tint-soft` (13% light / 18% dark), `--on-tint`. Charts use a 6-step sequential scale of the tint (6%, 20%, 38%, 58%, 80%, 100%).
- **Pipeline stage colours = Apple system colours**: New blue `#007aff`/`#0a84ff`, Contacted teal `#30b0c7`/`#40c8e0`, Quoted orange `#ff9500`/`#ff9f0a`, Deposit indigo `#5856d6`/`#5e5ce6`, Confirmed green `#34c759`/`#30d158`, Lost gray `#8e8e93`/`#98989d` (light/dark).
- Radius 12px (10px small). No card shadows; only floating layers get `0 10px 40px rgba(0,0,0,.18), 0 0 0 .5px rgba(0,0,0,.08)`.
- Components: translucent sidebar (desktop) with the full logo lockup and a small "PORTAL" label; translucent bottom **tab bar** on mobile (Today, Board, Tasks, Chat, More); segmented controls; inset grouped lists; alert-style dialogs; sheet-style side panel for a lead; **Spotlight-style ⌘K palette** (search leads, jump to screens, run actions); HUD toasts; 44pt minimum touch targets; visible focus rings; skip-to-content link; respects `prefers-reduced-motion`.
- Faded brand photo behind the workspace via `body::before` (opacity .09 light / .13 dark).
- Must work at 375px wide with no horizontal scroll and pass WCAG AA contrast.

## 5. Core screens (everyone)

**Lead source ("Came from").** Every lead records how the client first reached you: Website, Facebook, Instagram, Email, Guest referral, Agent or Other (a fixed list in one PHP constant, mirrored in JS). Leads from the website form default to Website; everything else starts as "Not set". For Guest referral, Agent and Other a second field holds the name (the guest who sent them, the agency, or e.g. the Facebook group). Each source has its own soft-tinted badge (brand-coloured Website, Facebook blue, Instagram pink, Email teal, Referral green, Agent indigo, with lighter text in dark mode). The badge appears next to the client's name everywhere: board cards, the Enquiries table, the lead header and the Emails conversation list. Changing it is logged on the lead ("Came from Facebook").

1. **Today** — an "@ Mentions" card when someone mentions you in chat, then KPI tiles with 14-day sparklines: New today (vs last week), Awaiting reply (red if any over the reply target), Follow-ups due (overdue count), Open pipeline value (quoted + deposit), Confirmed · 90 days, Median first reply (vs target). Below: "needs you now" list sorted by urgency, each lead showing a timer badge (ok / warn / bad by reply target and stale days).
2. **Board** — kanban with the six stages, drag and drop (and keyboard/menu move for accessibility), column totals in money, card shows name, source badge, trip, value, age badge. Moving to Lost asks for a reason.
3. **Tasks** — per-lead tasks plus general tasks, due dates, owner, done state. A booking **checklist template** in Settings auto-creates tasks relative to booking date / trip start / trip end.
4. **Calendar** — month view of booked trips (start–end), flags guide or vehicle double-bookings.
5. **Enquiries** — searchable table of every lead, filterable by stage, date range, type and **source**, with CSV export (includes source and referrer). Directors get row checkboxes and a Delete bar, plus a trash button on each lead: delete moves to the WordPress trash, with an Undo toast that restores it. Opening a lead shows the lead sheet: all form fields, stage, value, follow-up date, owner, **Came from** picker (+ referrer name), activity log, tasks, trip dates, guide, vehicle, costings, and **reply templates** (merge fields like {first_name}, {trip}) that open the email/WhatsApp with text filled in and log the first-reply time.
6. **Insights** — range segmented control (30d / 90d / 12m / all): Enquiries (vs previous period), Win rate, Confirmed revenue, Average trip value, Median first reply; charts: enquiries over time (current unfinished period drawn lighter), confirmed revenue by month, stage funnel, **where clients come from** (by lead source), **bookings by source** (deposit paid or confirmed, so you see which channel actually books), page the form was sent from, lost reasons, countries.
7. **Content** — newsletter subscribers list + CSV export.
8. **Settings** — reply templates, reply target, stale days, checklist template, theme, notifications, sample data (seed / remove).

## 6. Team Chat and Notes (everyone)

Both are add-on modules: their own PHP class, custom tables (created by `dbDelta` from a `maybe_install()` on `init`, keyed on a DB-version option, because file-upload deploys never run the activation hook), their own JS file registered as a portal extension, and REST under `{{prefix}}-portal/v1/chat/*` and `/notes/*`. Access is strict: you can only read a conversation you are a current member of, and a note you own or that is shared with you. Directors get no exception.

### Team Chat (iMessage-style)

- **Tables:** conversations (DM or group; a DM is unique per pair via a `dm_key` like `3:7`), members (role, `left_at`, `last_read_id`, `last_read_at`, `cleared_id`), messages (text, `reply_to`, `lead_id`, files JSON, mention ids, client id `cid`, edited/deleted times), reactions (one per person per message), files, and a **change counter** table.
- **Sync, not just "after_id":** every change (new message, edit, delete, reaction, read, membership) takes the next number from the counter. The client polls `GET chat/sync?rev=N` and gets everything changed since N in one request: changed conversations, messages, conversations it left, unread total, unread @mentions, and for the open thread the members' read positions and who is typing. Poll every 4 s with Chat open, 30 s elsewhere, back off on errors, pause in a hidden tab. Keep the transport behind one small object so WebSockets can replace polling later. History pages with `before_id` / `around`.
- **Layout:** conversation list | thread on a computer; on a phone the list, then a full-screen thread with a back button (tab bar hidden), sized to `visualViewport` so the composer sits above the keyboard. Put **Chat in the phone tab bar** (Calendar moves to More) with an unread badge.
- **Messages:** your bubbles on the right in the brand tint, others' on the left in light grey (`#e9e9eb` / dark `#2c2c2e`); tails only on the last bubble of a run; runs within 5 min grouped (one avatar, name above in groups); day separators "Today", "Yesterday", weekday, "Mon 9 Nov"; emoji-only messages shown large without a bubble.
- **Composer:** Enter sends (Shift+Enter new line; on touch devices Return is a new line), `@` opens a member picker and stores mention ids, `#` searches enquiries and attaches a lead card (name, trip dates, stage) that opens the lead, emoji palette, paste or attach photos/files, reply-to with a quoted line, Up arrow edits your last message.
- **Optimistic and offline-safe:** a sent bubble appears at once with a client id; the server treats a repeated `cid` as the same message, so retries never post twice; unsent messages wait in a localStorage outbox and show "Not delivered · Tap to retry".
- **Tapbacks** 👍 ❤️ 😂 ‼️ ❓ from a menu on hover (desktop) or long-press (phone); **edit within 15 minutes** ("Edited"); **delete** really wipes the text and files and shows "Message deleted".
- **Receipts:** "Delivered" / "Read 10:42" in DMs, "Seen by 3" (names on hover) in groups. **Typing dots** from a short-lived transient, pinged at most every 3 s.
- **Groups:** optional name, add people, the creator (or a Director) removes people, anyone can leave; grey system lines ("George added Hassan").
- **Deleting conversations** (from the conversation's ⋯ details), as in Messages: **Delete conversation** (anyone) stores the last message id in your member row's `cleared_id`; sync, history, Discussions and Claude's tools then hide everything up to it, and the conversation leaves your list until someone writes again, when it returns with only the new messages. Everyone else keeps their copy. **Delete for everyone** (Directors only, confirmed first) removes the messages, reactions, files and the conversation row, and marks every member as left so their next sync drops it.
- **Voice notes:** when the composer is empty the send button becomes a mic. Record with `MediaRecorder`, preferring `audio/mp4` (plays on iPhones) over WebM/Opus; show a recording bar with bin, red dot, timer and live level bars from an `AnalyserNode`; max 5 minutes. Upload with the duration and a ~36-bar waveform; the server checks the container from the file's magic bytes (WebM / MP4 `ftyp` / Ogg), not the browser's label. Playback bubble: play/pause, waveform that fills as it plays, countdown. **The file endpoint must answer HTTP `Range` requests with 206**, or Safari will not play audio.
- **Files are private:** stored outside the media library in a deny-all folder with random names, served only through `chat/file/{id}?_wpnonce=…` after a membership check, with `X-Content-Type-Options: nosniff` and a sandboxing CSP. Images over 1,600 px are resized. Logged-out requests get 401, direct links 403.
- **Lead link:** each enquiry page gets a **Discussions** card listing linked messages (only from conversations you are in) and a "Discuss in chat" button.
- **Mentions:** a mention sends a push even if the chat is muted, and an "@ Mentions" card appears on Today until read.
- Rate-limit sending (30 a minute per person). Don't push to someone who is looking at that conversation right now.

### Notes (Apple Notes-style)

- **Tables:** folders (per person), notes (owner, folder, lead, title, preview, sanitised HTML body, plain text for search, pinned, locked, lock hash, `version`, created/updated/updated_by, `deleted_at`), shares (note, user, can_edit).
- **Layout:** Folders | List | Note on a computer; one screen at a time on a phone. Smart folders: All Notes, Pinned, Shared with me, Recently Deleted (restore, "Delete now", purged after 30 days by a daily cron). Folder counts; deleting a folder moves its notes to the default "Notes" folder.
- **List:** title = first line, date (time today, weekday this week, else dd/mm/yy) and second line; pinned on top; sort by edited / created / title; search all titles and text (server-side `LIKE`, snippet around the match).
- **Editor:** a `contenteditable` page on a warm paper colour (`#fffdf8`, dark `#1c1c1e`), first line styled as the title. Toolbar: Title / Heading / Subheading / Body / Monospaced, bold, italic, underline, strikethrough, checklist (tickable circles; "move ticked to bottom"), bullets, numbers, table (add/delete rows and columns), photo, link. Set `defaultParagraphSeparator` to `p`; paste as plain text; lift lists and tables out of paragraphs after each command so the HTML stays valid; never put a table inside a list.
- **Autosave** 1 s after typing stops ("Saving…" / "Saved" / "Offline · will save"), with a local draft copy until the server confirms. Never redraw the editor while it has focus; make the app's global re-render skip when a `contenteditable` is focused.
- **Versions, not overwrites:** every body save sends the version it started from; if it is stale the server returns 409 with the current note and the editor offers "Keep mine" / "Use theirs". Show "Edited by Name · date".
- **Sanitise on the server** with a strict `wp_kses` allowlist (headings, paragraphs, inline formatting, lists with `ul.checklist` and `li[data-checked]`, tables, `img[data-file]` only, links with http/https/mailto/tel). Strip script/style/iframe blocks with their contents. Images are referenced by file id, never by URL, and attached to the note on save so readers of a shared note can see them.
- **Sharing:** private by default; share with chosen staff as view or edit. Only the owner pins, moves, links, shares, locks or deletes.
- **Lock note:** password checked with `wp_hash_password`, and the body **encrypted** with AES-256-GCM using a PBKDF2-derived key (per-note salt), so it is unreadable in the database and backups. Warn clearly that a forgotten password cannot be recovered. Locked notes keep the title visible, lose their shares, are left out of search, re-lock when you leave them, and pause after 5 wrong passwords.
- **Lead link:** attach a note to an enquiry; each enquiry page gets a **Notes** card with "New note".

## 7. Director-only modules (pick what you want)

- **Payments** — invoices and payments per lead, sequential invoice numbers, business details for the invoice header, public `/invoice/{token}/` page; KPIs: received this month, received this year, still to collect, overdue invoices.
- **Proposals** — itinerary proposal builder per lead, public `/proposal/{token}/` page with an Accept button (REST), KPIs sent / opened / accepted.
- **Emails** — send branded HTML email from the portal with wp_mail (SMTP), optional IMAP inbox fetch every 10 minutes (read-only, files each reply on the matching lead), history saved on the lead. Screen has three tabs: **Conversations** (one row per lead with emails, unread dot, source badge, message count, last subject and snippet, time ago), **Other inbox** (mail from people with no lead) and **Sending & inbox settings** (from name/address, reply-to, signature, test send, mailbox credentials stored encrypted).
- **Guests** — guest book built from all leads (trips, booked yes/no, newsletter consent) and newsletter campaigns sent in batches by WP-Cron, with unsubscribe handling.
- **Edit website** — safe editors for the site's own content: page text/photo editor, package/product create (copy as draft), hide, delete, price; find & replace limited to text, links and SEO; contact details stored in one option and used site-wide; top bar + announcement. Back up every write (keep the last 15 per page).
- **Visitors** — first-party, cookieless analytics: a small beacon in wp_footer posts to a public REST `hit` endpoint (same-origin check, skip bots and logged-in staff, honour DNT/GPC). Store hits in a custom table, prune after 400 days by daily cron. Visitor key = daily-salted hash, never store IPs. Country from the browser time zone. KPIs: visits, pages per visit, time on page, left after one page, opened enquiry form; top pages, sources, countries, devices, form funnel.
- **Suppliers & costing** — quote a trip in minutes. Directors keep suppliers (private CPT: lodges, camps, hotels, vehicle hire, flights, activities) with their own seasons, room types and meal plans, priced either *net + markup %* or *rack − commission %*; a park-fee table (resident / non-resident, adult / child, per-vehicle fees); the company fleet with daily costs; and default markup and minimum-margin settings. Anyone can build a **costing** on a lead: pick a lodge per night, vehicles, flights and extras; a pure-PHP pricing engine (no WordPress calls, unit-testable from the CLI) finds the season for each night, applies park fees and returns totals, per-person price, cost, profit, margin and problems ("No rate for {date}"). Costings are versioned (draft → review → final → replaced). Planners only ever receive selling prices (cost data is stripped server-side) and press *Ready for review*; Directors finalise, which snapshots the priced result and can create the proposal and deposit invoice in one click.
- **Ledger feed** — read-only JSON feed `GET /wp-json/{{prefix}}-portal/v1/ledger-feed` authenticated with an `X-Ledger-Key` header (generate / regenerate / disconnect in Settings), `since` and `include_drafts` parameters, `Cache-Control: no-store`.

## 8. Connect Claude (MCP)

- An MCP server inside the plugin: `POST /wp-json/{{prefix}}-portal/v1/mcp`, Streamable HTTP, JSON-RPC 2.0, no SSE. Every write runs through the same update function as the UI, so it is logged on the lead under the acting user.
- Tools: pipeline_summary, list_leads, get_lead, update_lead (stage + lost reason, value, follow-up, owner, note, **source** and referrer), create_lead (for leads from phone, WhatsApp, Facebook groups etc., with source), add_task, complete_task, open_tasks, general_task, set_trip, delete_lead (Directors only), invoice, payment, proposal, send_email, list_guests, list_staff, list_conversations, read_messages (respects "Delete conversation"), send_chat_message (by conversation or straight to a staff member as a DM), list_notes, get_note, create_note (plain text or simple HTML, folder, lead), update_note (replace or append; pass the version to avoid overwriting newer edits).
- Two ways to sign in: a Bearer key (constant in wp-config.php or an option, never in the repo) for Claude Code / desktop, and **OAuth 2.1** for Claude on the web and phone: discovery documents at `/.well-known/oauth-authorization-server` and `/.well-known/oauth-protected-resource`, dynamic client registration, an approval page behind the normal login, authorization code with PKCE S256, refresh tokens. Accept only Claude's and local callback URLs; store codes and tokens as SHA-256 hashes; access tokens last 1 hour, refresh tokens 60 days and rotate on use.

## 9. PWA and notifications

- Serve `/portal/manifest.webmanifest` and `/portal/sw.js`, app icons from the brand icon, "Install app" button.
- Web Push with VAPID keys stored in an option (pure PHP OpenSSL, aes128gcm). Events: new enquiry (everyone), reply and proposal accepted (Directors), 08:00 daily digest, team chat message and @mention (to that person only). Per-user on/off preferences and a "send test" button. Mute push while seeding sample data.

## 10. Sample data and handover

- A seeder that creates ~45 realistic sample enquiries across all stages and dates, ~25 subscribers and sample visitor hits, all flagged as demo, plus a one-click "Remove sample data" in Settings.
- Before calling it done: load every screen in light and dark mode at desktop and 375px width, move a demo lead on the board and back through the real REST API to prove saves work through the host's cache, check the browser console is clean, then send me screenshots.
- Do not touch the public enquiry form, the theme, or other plugins. Back up any existing file you must edit.
