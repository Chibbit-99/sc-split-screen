# Split Screen

A lightweight, single-page split-screen viewer. Load two websites side by side using URL parameters, resize the divider, and switch between light and dark themes.

## Features

- Two independent iframe panes.
- URL parameters to choose the left and right pages.
- Automatically adds `https://` to bare hostnames such as `flow.scspace.app`.
- Adjustable divider width (0–32 px).
- Drag the centre divider to change the left/right pane proportions. You can also focus the divider and use the left/right arrow keys.
- Back button beside the title that opens the left-side URL in the current tab.
- Light/dark theme toggle.
- Persists the selected theme in local storage.
- Broadcasts theme changes to the embedded pages with `window.postMessage()`.

## Usage

The page accepts two query parameters:

| Parameter | Description |
| --- | --- |
| `l` | URL to display in the left pane. |
| `r` | URL to display in the right pane. |

Example:

```text
https://YOUR-SPLIT-SCREEN-HOST/?l=https%3A%2F%2Fflow.scspace.app&r=https%3A%2F%2Fgithub.com
```

You can also use bare hostnames:

```text
?l=flow.scspace.app&r=github.com
```

Bare hostnames are automatically treated as HTTPS URLs. Full URLs with a scheme are also supported.

The values are URL query parameters, so encode URLs when they contain their own query strings, ampersands (`&`), or fragments (`#`). In JavaScript, use `encodeURIComponent(url)` when building a link.

If either parameter is omitted or is not a valid HTTP(S) URL, that pane remains on its placeholder. The other pane can still load normally.

## Controls

### Back button

The arrow button beside **Split Screen** navigates the current tab to the URL supplied in `l`. It is disabled when there is no valid left URL.

### Divider

Use the **Divider width** slider to set the centre divider from 0 to 32 pixels. You can also drag the divider to adjust the relative width of the two panes. When the divider is keyboard-focused, the left and right arrow keys adjust the split.

### Theme

The theme button switches the split-screen page between light and dark mode. The selected value is saved in local storage under:

```js
localStorage.getItem("split-screen-theme")
```

The stored value is either `"light"` or `"dark"`. If storage is unavailable or there is no saved value, the page defaults to light mode.

The theme setting belongs to the split-screen page's origin. It is shared by split-screen pages on that same origin, but is not shared with the websites loaded inside the iframes.

## Iframe dark-mode messaging

The split-screen page does **not** try to reach into an iframe and edit its document. Browsers restrict that kind of direct access for cross-origin pages. Instead, it sends a message to each embedded page using the cross-origin `postMessage()` API.

### Message format

The message is a plain string, not an object:

| Theme | Message |
| --- | --- |
| Dark | `"dark-theme"` |
| Light | `"light-theme"` |

The message is sent to each configured iframe's `contentWindow`.

### When messages are sent

1. On startup, the split-screen page reads `split-screen-theme` from local storage and applies the saved theme to its own interface.
2. It starts loading the configured iframe URLs.
3. It waits until **every iframe with a valid URL parameter** has fired its `load` event.
4. Once all configured frames have loaded, it broadcasts the current theme message.
5. When the user changes the theme, the new value is saved to local storage first, the split-screen UI is updated, and the new theme message is broadcast if all configured frames have loaded.
6. If an iframe loads or navigates again, its `load` event causes the current theme to be broadcast again once all configured frames have loaded.

An omitted or invalid pane is not counted as an iframe that needs to load. If both panes are empty, there are no iframe recipients.

The sender uses:

```js
const message = theme === "dark" ? "dark-theme" : "light-theme";
iframe.contentWindow.postMessage(message, "*");
```

The target origin is `"*"` because the iframe destinations are supplied dynamically and can be on different origins. The receiving page should therefore validate the sender before acting on a message.

### Listening for theme messages

Add a message listener to the website that will be embedded. Replace `https://YOUR-SPLIT-SCREEN-HOST` with the exact origin hosting this split-screen page (scheme and host, without a path).

```js
const trustedParentOrigin = "https://YOUR-SPLIT-SCREEN-HOST";

window.addEventListener("message", (event) => {
  // Only accept messages from the split-screen page hosting this iframe.
  if (event.origin !== trustedParentOrigin) return;
  if (event.source !== window.parent) return;

  if (event.data === "dark-theme") {
    document.documentElement.dataset.theme = "dark";
  } else if (event.data === "light-theme") {
    document.documentElement.dataset.theme = "light";
  }
});
```

Then define the page's own light and dark styles. For example:

```css
:root {
  color-scheme: light;
  --page-background: #ffffff;
  --page-foreground: #141414;
}

:root[data-theme="dark"] {
  color-scheme: dark;
  --page-background: #121212;
  --page-foreground: #f2f2f2;
}

body {
  background: var(--page-background);
  color: var(--page-foreground);
}
```

The split-screen page only sends the message; the embedded website must implement the listener and its own dark-mode styles. A website that does not listen for these messages will not change theme automatically.

**Important:** Register the message listener early in the embedded page's lifecycle so it is ready when the iframe finishes loading. The sender waits for iframe `load` events before its initial broadcast, which gives the embedded page time to install its listener.

## Browser and embedding limitations

Some websites prohibit being displayed inside an iframe using headers such as `X-Frame-Options` or a Content Security Policy `frame-ancestors` directive. Those sites may refuse to load here. This page cannot bypass those browser security restrictions.

Cross-origin pages can receive `postMessage()` events, but the parent page cannot directly inspect or modify their DOM. Theme support must be implemented by the embedded website itself.

## Running / hosting

This is a static HTML page; there is no build step or server-side component required. Serve `index.html` from any static web host, then open it with the `l` and `r` parameters.

For example, a local static server can serve the file during development:

```bash
python3 -m http.server 8000
```

Then visit:

```text
http://localhost:8000/?l=flow.scspace.app&r=github.com
```
