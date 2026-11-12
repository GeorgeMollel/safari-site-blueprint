---
name: operator-portal
description: Build a private operations portal (WordPress plugin, single-page app at /portal/) for a tour or service business, in Apple Human Interface Guidelines style. It includes Today KPIs, a kanban lead board, tasks, a trip calendar, enquiries with lead sources (website, Facebook, Instagram, email, guest referral, agent), reply templates and insights, iMessage-style team chat (groups, voice notes, tapbacks, read receipts, @mentions, enquiry links, delete for me or for everyone) and Apple Notes-style notes (folders, checklists, tables, sharing, encrypted locked notes), plus Director-only payments, proposals, email conversations, guests/newsletter, suppliers and costing, a website editor, cookieless visitor analytics, a ledger feed, an MCP server with OAuth so Claude can work the pipeline, PWA and web push. Use when asked for a lead/booking dashboard "like the Asili Portal" or a CRM-style back office on a WordPress site.
---

# Operator portal (Asili Portal pattern)

The full specification is in `reference.md` (next to this file). Read it first and ask which optional Director modules the user wants.

## Workflow

1. **Find the lead source.** Work out where enquiries already land (CPT, form plugin entries, email). Read from it and never change the public form.
2. **Scaffold the plugin.** Main file with a `VERSION` constant, `includes/class-*.php` per module, `views/shell.php`, one CSS file and one vanilla JS file, a `/portal/` rewrite and the REST namespace `{prefix}-portal/v1`.
3. **Roles.** Create a Planner role with the `{prefix}_portal` capability (kept out of wp-admin) and label Administrators "Director". Restyle the login page.
4. **Design tokens.** Copy the HIG token set from the spec exactly: one brand tint, stepped for contrast; Apple system colours for stages; dark mode as a token swap on `:root`.
5. **Core screens**, in this order: Today → Board → Enquiries (lead sheet + reply templates) → Tasks → Calendar → Insights → Settings. Then Team Chat, then Notes (each its own class, tables and JS file).
6. **Sample data.** Seed demo data flagged `_{prefix}_demo=1` so every screen can be checked, with a one-click remove.
7. **Optional modules** the user picked, then the MCP connection, then the PWA and push.
8. **Deploy and verify.** Bump `VERSION`, deploy, check light/dark at desktop and 375px, and prove a save works through the host cache with a real REST round trip. Share screenshots.

## Rules

- Every write goes through REST with nonce + capability checks, with an optimistic UI that rolls back and shows a toast on error.
- Visitor analytics: no cookies, no IPs, daily-salted hashes, respect DNT/GPC.
- Never store keys or passwords in JS or in the repo. Keep VAPID, ledger and MCP keys in options or wp-config.php.
- Record a source on every lead and show it as a badge wherever the client's name appears, so the owner can see which channel brings bookings.
- Planners never receive supplier cost, commission or margin; strip it on the server, not in the UI.
- Chat and Notes: membership/ownership checks on every route (no Director exception), private file storage served through a permission-checked endpoint that supports HTTP Range, a change counter for chat sync, a version check (409) on every note save, and a strict kses allowlist for note HTML.
- Create custom tables from a versioned `maybe_install()` on `init`, not only on activation: deploys that upload files never run the activation hook.
