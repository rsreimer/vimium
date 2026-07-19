# Custom-command page-world execution

## Decision

Use `chrome.userScripts.execute()` to run a saved command body on demand in
the `MAIN` world. It is the Chrome-supported API for arbitrary user-provided
code and supports a JavaScript source string. Wrap each saved body in an async
IIFE before passing it as `js: [{code: ...}]`, for example:

```js
(() => (async () => {
  // saved command body
})())()
```

`execute()` waits for a returned promise to settle, so the body may use
`await`. Target the current tab's top frame unless a later product decision
explicitly adds frame selection.

## API constraints

- `chrome.scripting.executeScript()` is not a suitable runtime-code API: it
  accepts an extension file or an extension-defined `func`, not a JavaScript
  string. Its function is serialized/deserialized, losing its closure, bound
  parameters, and execution context; `args` must be JSON-serializable.
  [Chrome scripting API](https://developer.chrome.com/docs/extensions/reference/api/scripting)
- `chrome.userScripts` is expressly the API for code provided by a user that
  cannot be packaged with the extension. `UserScriptInjection.js[].code`
  accepts the source string needed for a saved command.
  [Chrome userScripts API](https://developer.chrome.com/docs/extensions/reference/api/userScripts)
- `MAIN` is the DOM JavaScript environment shared with the host page. This is
  what makes page globals available, but it also makes the command visible to
  and potentially interfered with by page code and other extensions. Treat
  this execution context as untrusted and do not expose extension privileges
  through the wrapper.
  [Chrome userScripts API](https://developer.chrome.com/docs/extensions/reference/api/userScripts)

## Compatibility and permission gate

`userScripts` requires the `userScripts` manifest permission and host
permission for every target page. The API is available in MV3 from Chrome 120;
on Chrome before 138, users must enable Developer mode, while Chrome 138+ uses
an extension-specific **Allow User Scripts** toggle. The API can be unavailable
when that toggle is off; Chrome documents probing `getScripts()` to detect it.

Most importantly, on-demand `chrome.userScripts.execute()` is Chrome 135+.
Vimium currently declares a minimum Chrome version of 117, so this capability
must be feature-gated with a clear unavailable state on Chrome/Chromium below
135 (or the whole extension minimum must be raised). `register()` is not a
substitute: registered user scripts run on matching page loads, whereas the
feature needs key-triggered execution. Registered user scripts are also cleared
on extension update.

[Chrome userScripts API](https://developer.chrome.com/docs/extensions/reference/api/userScripts)

## CSP and validation consequences

Chrome documents that the `USER_SCRIPT` world is exempt from the page CSP and
has a configurable CSP. It does **not** make that exemption for `MAIN`.
Therefore, MAIN-world command compatibility with restrictive site CSP must be
verified in a Chromium integration test/proof of concept before it is promised;
do not claim CSP bypass based on the user-scripts documentation.

Do not validate a command body by calling `eval` or `new Function` in Vimium's
options page or service worker. MV3 extension-page CSP forbids evaluating
strings as executable code, and Chrome does not permit relaxing that policy
with `unsafe-eval`. Use a parser bundled with Vimium (or validation supplied by
the supported user-scripts flow) instead.

[Chrome userScripts API](https://developer.chrome.com/docs/extensions/reference/api/userScripts)
[Chrome extension CSP](https://developer.chrome.com/docs/extensions/reference/manifest/content-security-policy)

## Consequences for implementation

1. Add `userScripts` to the manifest and retain Vimium's existing `<all_urls>`
   host permission; provide onboarding/detection for the browser toggle.
2. Have the existing content-side key handler ask the service worker to execute
   the selected command in its tab. The service worker owns the privileged API
   call; the saved source never becomes extension code.
3. Generate one fixed async wrapper around the body and pass only that string
   to `chrome.userScripts.execute({target: {tabId}, world: "MAIN", js: [...]})`.
   Its rejected promise should be caught and logged by the caller so the agreed
   DevTools-only runtime-failure behavior is preserved.
4. Test at least a page with a strict CSP, a command that accesses a page
   global, a rejected async command, a disabled user-scripts toggle, and a
   Chrome version below 135.
