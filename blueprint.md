# blueprint.md — MailBlueprint Technical Specification

## 1. Purpose
An Outlook alternative: a program that gathers multiple email accounts (general-purpose IMAP/SMTP) into a single Windows desktop app. Target user accounts are cPanel/Roundcube-based hosting accounts (typical: host `mail.<domain>.com`, IMAP 993 SSL / 143, SMTP 465 SSL / 587 STARTTLS). NO dependency on a specific provider API (Gmail API, MS Graph); manual host/port/user/password entry.

## 2. Technology
- **Electron** (^31), plain HTML/CSS/JS renderer (no framework)
- `imapflow` (IMAP), `nodemailer` (SMTP), `mailparser` (parsing), `electron-store` (persistent data)
- `electron-builder` (^24) targeting `nsis` for a Windows `.exe`
- package.json scripts: `start: electron .`, `build: electron-builder --win`, `build:dir: electron-builder --win --dir`
- **Do not reference an icon** in the electron-builder config (no icon file exists; referencing one breaks the build). `files: ["src/**/*", "package.json"]`, output `dist/`.
- nsis config must include `createDesktopShortcut: true` and `createStartMenuShortcut: true` — the deliverable is a real installed app you open with a double-click, not something run from a terminal.

## 3. File structure
```
package.json
src/
  main.js          -> window, account CRUD, ALL IMAP/SMTP work, notifications
  preload.js       -> contextBridge bridge
  renderer/
    index.html     -> 3 panes + 2 modals
    style.css      -> dark theme
    renderer.js    -> UI logic
```

## 4. UI (Turkish language, dark theme)
Three panes, with draggable resizers between them (widths persisted to localStorage):
- **Left**: "My Accounts" list (+ add button, ✕ remove on each row), the selected account's "Folders" list, a "✎ New Mail" button at the bottom.
- **Middle**: header (account — folder), a refresh ⟳ button, a search box (search on Enter, ✕ to clear), the mail list (from/subject/date; unread = bold subject; a 🗑 delete button appears on hover).
- **Right**: reading pane. "↩ Reply" and "🗑 Delete" buttons at the top, subject, From/To/Date meta block, body, clickable attachment chips (filename + size) at the bottom.
- **Add account modal**: display name, email; IMAP host/port(993)/user/password/SSL checkbox; SMTP host("blank = IMAP host")/port(465)/user/password("blank = IMAP password")/SSL checkbox; a "Skip certificate validation" checkbox; "Test Connection" (shows IMAP and SMTP results on separate lines), Save, Cancel.
- **Compose modal** (wide): from-account selector, To/CC/BCC/Subject, a formatting toolbar (Bold/Italic/Underline/Bullet list/Numbered list/Link — via `document.execCommand`, buttons call `preventDefault()` on `mousedown` to keep editor focus), a `contenteditable` editor, "📎 Attach File" + attachment chips (removable with ✕), Send/Cancel.
- A toast that fades out in the bottom-right for delete/send results.

## 5. IPC contract (preload → main)
| Channel | Input | Output |
|---|---|---|
| `accounts:list` | – | account list without passwords |
| `accounts:add` | form data | updated list (password encrypted via `safeStorage`) |
| `accounts:remove` | id | updated list (also closes any open IMAP connection) |
| `accounts:test` | form data | `{ok, imap:{ok,error}, smtp:{ok,error}}` — SMTP test via `transporter.verify()` |
| `mail:folders` | accountId | `[{path, name, specialUse}]` |
| `mail:list` | accountId, folder, limit=50 | envelope summary of the last N messages (uid, subject, from, to, date, seen, size), newest first |
| `mail:search` | accountId, folder, query | `client.search({or:[{from:q},{subject:q}]}, {uid:true})` → fetch by uid, sort by date |
| `mail:read` | accountId, folder, uid | parsed message (subject, from, fromAddress, to, date, text, html, attachments[{index,filename,size,contentType}]) + adds `\Seen` on the server |
| `mail:delete` | accountId, folder, uid | `{permanent:bool}` — trash logic in §7 |
| `mail:saveAttachment` | accountId, folder, uid, index, filename | save dialog + `shell.showItemInFolder` |
| `compose:pickFiles` | – | `dialog.showOpenDialog` (multi-select) → `[{path,name,size}]` (do NOT rely on `File.path` in the renderer — Electron is removing it) |
| `mail:send` | accountId, to, cc, bcc, subject, text, html, attachments | `{ok, messageId, savedToSent}` — Sent copy per §8 |
| `mail:new` (main→renderer event) | – | `{accountId, count}` — renderer refreshes the list if INBOX is open |

In main, every handler goes through a single `handle(channel, fn)` wrapper: errors are caught, translated to Turkish via `trError()`, and re-thrown.

## 6. IMAP connection management
- A `Map<accountId, ImapFlow>` cache; reconnect if `client.usable` is false.
- Every mailbox operation: `const lock = await client.getMailboxLock(folder); try {...} finally { lock.release(); }`
- Mail list: over `mailbox.exists`, fetch the range `max(1, total-limit+1):total` with `envelope+flags+uid+size`, then reverse the array. Do NOT download bodies for the list.
- On `window-all-closed`, call `logout()` on all connections.

## 7. Delete logic
1. Find the Trash folder: first `specialUse === '\\Trash'`, otherwise a path/name regex (`trash|deleted items|çöp`).
2. If the message is not already in Trash → `messageMove(uid, trashPath, {uid:true})`, result "moved to trash".
3. If already in Trash, or no Trash folder exists → `messageDelete(uid, {uid:true})`, result "permanently deleted".

## 8. Sending and the Sent-folder copy (CRITICAL)
SMTP only delivers the mail; **copying it into Sent is the client's job** (Roundcube does this itself, which is why users assume it's automatic). Flow:
1. Send via `transporter.sendMail(mailData)`.
2. `new MailComposer({...mailData, messageId: info.messageId}).compile()` — `require('nodemailer/lib/mail-composer')`; set `compiled.keepBcc = true` (so BCC recipients remain visible in the copy); `const raw = await compiled.build()`.
3. Find the Sent folder (`specialUse '\\Sent'`, otherwise a name regex `sent|gönderilmiş`), then `client.append(sentPath, raw, ['\\Seen'], new Date())`.
4. If the append fails, the send is STILL considered successful; return `savedToSent:false` and let the UI inform the user.
- Attachments are passed to nodemailer as `[{path, filename}]`.
- Validation: only "To" is required; **Subject may be empty**.

## 9. Rendering HTML mail (CRITICAL)
- `<iframe class="mail-frame" sandbox="allow-same-origin" srcdoc="...">` — `allow-scripts` is NEVER granted. The srcdoc value is escaped with a dedicated `escapeHtmlAttr` (`&`, `"`, `<`).
- **Height**: a sandboxed iframe does not auto-size; in CSS, `.mail-frame { display:block; width:100%; min-height:300px; border:none; background:white; }`, and in JS, on the `load` event measure `contentDocument.documentElement.scrollHeight` and set `frame.style.height`; re-measure after 300ms and 1200ms since images load late. (`allow-same-origin` is required precisely for this measurement.)
- **Narrow-content problem**: many emails use fixed width/overflow in their own HTML. `makeResponsiveHtml(html)`: parse with `DOMParser`, give `body` an `id="__mailapp_wrap"`, and append a high-specificity `<style>` (`#__mailapp_wrap * { max-width:100% !important; ... }`) to the END of `head`.
- **CSP**: add a meta CSP to `index.html` (`script-src 'self'`; `img-src 'self' https: http: data: cid:`). A srcdoc iframe inherits the parent page's CSP — if `img-src` doesn't allow https, remote images in emails will not render.

## 10. Background new-mail check
- First pass 15 seconds after launch (to learn baseline uidNext values, does not fire a notification), then every 2 minutes.
- For each account, `client.status('INBOX', {uidNext:true})`; if greater than the previous uidNext, show a Windows notification for the difference ("N new mail(s) arrived") (`Notification`, clicking brings the window to front) + emit `mail:new` to the renderer.
- Errors are swallowed silently.

## 11. Error translation (`trError`)
If the message already contains Turkish characters (it's one of our own messages), pass it through unchanged. Otherwise lowercase it and match patterns:
- `enotfound|eai_again|getaddrinfo` → server address not found
- `econnrefused` → suggest checking port/SSL
- `etimedout|timeout` → timed out
- `certificate|self signed|unable to verify|cert` → point to the "Skip certificate validation" option
- `wrong version number|ssl routines` → port↔SSL mismatch (993/465 need SSL on, 143/587 need it off)
- `auth|535|invalid credentials|login|password` → wrong username/password
- no match: "Operation failed: <raw message>"

Self-signed support: an `allowSelfSigned` flag on the account → pass `tls: {rejectUnauthorized:false}` to both imapflow and nodemailer.

## 12. Debug hook
In `main.js`, if `process.env.MAILAPP_DEBUG` is set, pipe the renderer console to stdout via `webContents.on('console-message')`. Smoke test: launch with that variable, wait ~10s, confirm no errors in the output, then close the process.

## 13. Lessons Learned
1. **"My sent mail doesn't show up in Sent"** → SMTP doesn't save a copy; the IMAP APPEND in §8 is mandatory.
2. **"The mail renders in a tiny box"** → the CSS wasn't targeting the iframe (the selector was written for a container the iframe wasn't actually inside), and a sandboxed iframe doesn't auto-size; the class + height measurement in §9 is mandatory. Double-check that CSS selectors actually match the DOM you produced.
3. **Electron's CSP security warning** → resolved by adding the meta CSP; but narrowing `img-src` breaks images in emails (§9).
4. **`Error invoking remote method 'x': Error: ...` prefixes** → stripped in the renderer with a regex; the user must never see the raw IPC prefix.
5. **Requiring a subject** annoyed the user → only "To" is required.
6. **Self-signed certificates on cPanel** are a real-world scenario → per-account `allowSelfSigned` + guidance in the error message.
7. **Race condition opening two mails in quick succession** → by the time `mail:read` resolves, the user may have switched to another message; check that `currentUid` still matches before rendering.
8. **File picking on Windows** → don't rely on the renderer's `File.path`; run `dialog.showOpenDialog` in main instead (§5).
9. **"Not a real desktop app"** → the app must not be delivered as something the user launches with `npm start` in a terminal. Build the NSIS installer and run it so a Desktop shortcut and Start Menu entry actually get created; that's the finished, double-click-to-open deliverable.
