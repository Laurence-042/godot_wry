# Caveats

Here are the main limitations you should know about before using Godot WRY.

## Webview always renders on top

The webview is rendered directly in the OS window as a native UI element, always appearing on top of your game content. You can't render it on 3D meshes or have game elements appear over it. Think of it as a desktop application overlay rather than an in-world UI element.

## Different browser engines across platforms

Since each platform uses its native webview (WebView2, WebKit, WebKitGTK), your web content may behave differently across Windows, macOS, and Linux. Test on all target platforms, especially when using newer web features.

## No automatic dependency checks

The extension doesn't verify or install required dependencies like WebKitGTK on Linux. You're responsible for ensuring users have the necessary libraries installed and handling missing dependencies gracefully.

We are open for handling this on Godot WRY's side, so contributions are welcomed.

## Focus and the native webview overlay

Because the webview is a native OS overlay (see [Webview always renders on top](#webview-always-renders-on-top)), it sits **outside** Godot's normal input and focus system. Two properties control how focus behaves, and they answer different questions:

| Property | Answers | Default |
| --- | --- | --- |
| [`focused_when_created`](/reference/webview#focused-when-created) | Should the web page grab native keyboard focus the moment it's created? | `true` |
| `focus_mode` (inherited from `Control`) | Should the WebView participate in Godot's GUI focus chain at all? | `None` |

**Why `focus_mode` matters here.** Clicks inside the native webview never reach Godot, so Godot cannot auto-transfer its GUI focus to the WebView on click. If `focus_mode` is `None`, the WebView stays out of Godot's focus chain entirely. If you set it to `Click`, clicking the WebView explicitly grabs Godot GUI focus away from whatever Control previously held it (and clicks inside forward a `grab_focus` internally). While the WebView holds Godot focus, keyboard events captured by the page are no longer forwarded back to Godot (see [`forward_input_events`](/reference/webview#forward-input-events)), which prevents a key like <kbd>Space</kbd> from activating both the page and a still-focused Godot button at once.

### Which combination should I use?

Think about whether the webview is the thing the user interacts with, or a layer over other Godot UI.

**The webview is interactive in-app content** (e.g. a form, a settings panel, a minigame) and other Controls also use keyboard input:

- Set `focus_mode` to **`Click`** (or `All` if you want Tab navigation in/out of the WebView). This stops keyboard events from double-firing: without it, a button the user just clicked still holds Godot GUI focus, so forwarded keys (e.g. <kbd>Space</kbd>) reach both the page and that button.
- Set `focused_when_created` to whatever initial focus you want: `false` if Godot should keep focus on startup (the user clicks into the page when ready), `true` if the page should be immediately typeable.

**The webview is a non-interactive overlay** (e.g. a HUD, a video player, a decorative layer) sitting on top of Godot UI that handles all the input:

- Leave `focus_mode` at **`None`** (the default). The WebView won't take part in Godot's focus chain, and keyboard events captured by the page are forwarded to Godot via `forward_input_events`.
- Set `focused_when_created` to `false` so the page doesn't grab native focus on startup and steal keystrokes from your Godot UI before the user interacts with it.

> The two properties are independent — `focused_when_created` only affects the moment of creation, while `focus_mode` governs ongoing focus behavior. In particular, `focus_mode: None` does **not** mean the page can never receive input; the native webview still takes focus when clicked, it just never tells Godot about it.
