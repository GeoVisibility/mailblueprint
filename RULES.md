# RULES.md — Non-Negotiable Rules

## Security
1. Passwords are **never written to disk in plain text**: encrypt with Electron `safeStorage` (DPAPI on Windows), store as a base64 string in `electron-store`. If `safeStorage` is unavailable, a base64 fallback is acceptable, but the encrypted path is the default.
2. Passwords are **never sent to the renderer**: when the account list is returned to the renderer, `imapPass`/`smtpPass` fields are stripped. All IMAP/SMTP operations happen in the main process.
3. `contextIsolation: true`, `nodeIntegration: false`; only a narrow API is exposed to the renderer via `contextBridge`.
4. HTML emails **can never execute scripts**: sandboxed iframe + `srcdoc`. `sandbox="allow-same-origin"` may be granted for height measurement, but `allow-scripts` is NEVER granted.
5. `index.html` carries a CSP meta tag: `script-src 'self'`, with `img-src ... https: http: data: cid:` allowed so remote images inside emails still render.

## Language and error handling
6. UI language and all messages are **Turkish** (the target users are Turkish-speaking).
7. Raw `err.message` / stack traces are **never shown in the UI**. A single translation function in the main process (e.g. `trError`) wraps every IPC handler and maps known error patterns (DNS, port, timeout, certificate, authentication, SSL version mismatch) to helpful Turkish messages. The renderer strips the `Error invoking remote method '...':` prefix that `ipcRenderer.invoke` errors carry.
8. A certificate error message points the user to the "Skip certificate validation" option in the account form (cPanel servers frequently use self-signed certificates).
9. Background jobs (new-mail checks) may fail silently; they must not throw errors at the user, and they retry on the next cycle.

## Behavior
10. Delete = move to the server's Trash folder first (`specialUse \Trash`, or find it by name if unset); if already in Trash, or no Trash folder exists, delete permanently. Every delete asks for confirmation first, and the result (moved / permanently deleted) is reported to the user.
11. Opening a mail marks `\Seen` on the server (so it also shows as read in other clients).
12. When sending, only "To" is required; **Subject is not required**.
13. Every sent mail is copied into the Sent folder via IMAP APPEND; if the copy fails, the send is still considered successful and the user is informed.
14. Convenience defaults when SMTP fields are left blank: SMTP host falls back to the IMAP host, SMTP password falls back to the IMAP password.

## Performance
15. Opening a folder fetches at most the last 50 messages (envelope+flags only; the body is downloaded only when a message is opened).
16. The IMAP connection is cached per account and reused (checked via `client.usable`); a new connection is not opened on every request.
17. Mailbox operations are locked with `getMailboxLock`, and `lock.release()` is always called in a `finally` block.

## Code
18. No framework: plain HTML/CSS/JS. Dark theme.
19. Any text coming from the user or the server is escaped before being written to the DOM (XSS).
20. All three panes are resizable, and the widths are persisted to `localStorage`.

## Packaging
21. The NSIS installer must create both a desktop shortcut and a Start Menu entry (`createDesktopShortcut: true`, `createStartMenuShortcut: true`) so the finished app launches with a double-click like any other installed Windows program.
