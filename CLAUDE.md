# CLAUDE.md — How to Work in This Repo

This repo is not an application — it is a **build specification**. Your job: build the app described in `blueprint.md` in this folder, from scratch, fully working end to end.

## Read first
1. `blueprint.md` — what to build (architecture, features, IPC contract, critical details)
2. `RULES.md` — non-negotiable rules (security, language, error handling)

The **"Lessons Learned"** section in the blueprint documents real pitfalls hit during actual development; do not repeat them.

## Build order
1. **Scaffold**: `package.json` (dependencies, `start`/`build` scripts, electron-builder nsis config), `src/main.js`, `src/preload.js`, `src/renderer/{index.html, style.css, renderer.js}`. Run `npm install`.
2. **Account management**: add/remove/list + encrypted password storage via `safeStorage` + a "Test Connection" action that checks IMAP **and** SMTP separately.
3. **Reading**: folder list, last-50-messages list (unread shown bold), sandboxed-iframe HTML rendering, mark `\Seen` on open, attachment download.
4. **Sending**: rich-text editor, CC/BCC, file attachments, **IMAP APPEND to the Sent folder after sending** (see the blueprint for details — skip this and sent mail won't show up in Sent).
5. **Delete + search + notifications**: move-to-trash logic, server-side search, new-mail check every 2 minutes + Windows notification.
6. **Packaging**: verify `npm run build` produces the NSIS `.exe`, then **run the installer** so the app actually gets installed with a desktop shortcut and a Start Menu entry (`createDesktopShortcut: true`, `createStartMenuShortcut: true` in the electron-builder config) — the deliverable is a double-click-to-open desktop app, not something launched via `npm start` in a terminal.

## Verification (at every stage)
- Run `node --check` on all three JS files to catch syntax errors.
- Launch the app and watch the renderer console: add a `console-message` listener in `main.js` gated behind `process.env.MAILAPP_DEBUG`, launch with that variable set during a smoke test, and confirm **zero errors/warnings**.
- If Electron's CSP security warning appears, confirm the CSP meta tag from the blueprint was added to `index.html`.
- You will not have a real IMAP account: ask the user for test account credentials, or walk the user through testing themselves. Verify everything that doesn't require a live connection yourself.
- After building the installer, run it and confirm a shortcut file now exists on the Desktop (and in the Start Menu) pointing at the installed app.

## Delivery criteria
- `npm install && npm start` launches cleanly (no console errors).
- `npm run build` produces an NSIS installer under `dist/`.
- Running the installer creates a desktop icon that opens the app on double-click.
- A Turkish `README.md` in the built app's repo: setup, account-adding guide (including the cPanel "Email Accounts → Connect Devices" tip), day-to-day usage.
- Every rule in `RULES.md` is satisfied.

## Communicating with the user
- Always write to the user in Turkish (the target users of the produced app are Turkish-speaking).
- If the user doesn't answer a clarifying question, proceed with the recommended default and state clearly what you assumed.
