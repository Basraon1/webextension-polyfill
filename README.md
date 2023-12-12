

```
var browser = require("webextension-polyfill");

browser.storage.local.set({
  [window.location.hostname]: document.title,
}).then(() => {
  browser.runtime.sendMessage(`Saved document title for ${window.location.hostname}`);
});
```

By using `require("webextension-polyfill")`, the module bundler will use the non-minified version of this library, and the extension is supposed to minify the entire generated bundles as part of its own build steps.

If the extension doesn't minify its own sources, it is still possible to explicitly ask the module bundler to use the minified version of this library, e.g.:

```javascript
var browser = require("webextension-polyfill/dist/browser-polyfill.min");

...
```

### Usage with webpack without bundling

The previous section explains how to bundle `webextension-polyfill` in each script. An alternative method is to include a single copy of the library in your extension, and load the library as shown in [Basic Setup](#basic-setup). You will need to install [copy-webpack-plugin](https://www.npmjs.com/package/copy-webpack-plugin):

```sh
npm install --save-dev copy-webpack-plugin
```

**In `webpack.config.js`,** import the plugin and configure it this way. It will copy the minified file into your _output_ folder, wherever your other webpack files are generated.

```js
const CopyWebpackPlugin = require('copy-webpack-plugin');

module.exports = {
  /* Your regular webpack config, probably including something like this:
  output: {
    path: path.join(__dirname, 'distribution'),
    filename: '[name].js'
  },
  */
  plugins: [
    new CopyWebpackPlugin({
      patterns: [{
        from: 'node_modules/webextension-polyfill/dist/browser-polyfill.js',
      }],
    })
  ]
}
```

And then include the file in each context, using the `manifest.json` just like in [Basic Setup](#basic-setup).

## Using the Promise-based APIs

The Promise-based APIs in the `browser` namespace work, for the most part,
very similarly to the callback-based APIs in Chrome's `chrome` namespace.
The major differences are:

* Rather than receiving a callback argument, every async function returns a
  `Promise` object, which resolves or rejects when the operation completes.

* Rather than checking the `chrome.runtime.lastError` property from every
  callback, code which needs to explicitly deal with errors registers a
  separate Promise rejection handler.

* Rather than receiving a `sendResponse` callback to send a response,
  `onMessage` listeners simply return a Promise whose resolution value is
  used as a reply.

* Rather than nesting callbacks when a sequence of operations depend on each
  other, Promise chaining is generally used instead.

* The resulting Promises can be also used with `async` and `await`, rather
  than dealt with directly.

## Examples

The following code will retrieve a list of URLs patterns from the `storage`
API, retrieve a list of tabs which match any of them, reload each of those
tabs, and notify the user that is has been done:

```javascript
browser.storage.local.get("urls").then(({urls}) => {
  return browser.tabs.query({url: urls});
}).then(tabs => {
  return Promise.all(
    Array.from(tabs, tab => browser.tabs.reload(tab.id))
  );
}).then(() => {
  return browser.notifications.create({
    type: "basic",
    iconUrl: "icon.png",
    title: "Tabs reloaded",
    message: "Your tabs have been reloaded",
  });
}).catch(error => {
  console.error(`An error occurred while reloading tabs: ${error.message}`);
});
```

Or, using an async function:

```javascript
async function reloadTabs() {
  try {
    let {urls} = await browser.storage.local.get("urls");

    let tabs = await browser.tabs.query({url: urls});

    await Promise.all(
      Array.from(tabs, tab => browser.tabs.reload(tab.id))
    );

    await browser.notifications.create({
      type: "basic",
      iconUrl: "icon.png",
      title: "Tabs reloaded",
      message: "Your tabs have been reloaded",
    });
  } catch (error) {
    console.error(`An error occurred while reloading tabs: ${error.message}`);
  }
}
```

It's also possible to use Promises effectively using two-way messaging.
Communication between a background page and a tab content script, for example,
looks something like this from the background page side:

```javascript
browser.tabs.sendMessage(tabId, "get-ids").then(results => {
  processResults(results);
});
```

And like this from the content script:

```javascript
browser.runtime.onMessage.addListener(msg => {
  if (msg == "get-ids") {
    return browser.storage.local.get("idPattern").then(({idPattern}) => {
      return Array.from(document.querySelectorAll(idPattern),

j
File an issue in this repository for API methods that support callbacks in Chrome *and*
Firefox but are currently missing from the "API metadata" file.

### Issues that happen only when running on Firefox

When an extension that uses this library doesn't behave as expected on Firefox, it is almost never an issue in this polyfill, but an issue with the native implementation in Firefox.

"Firefox only" issues should be reported upstream on Bugzilla:
- https://bugzilla.mozilla.org/enter_bug.cgi?product=WebExtensions&component=Untriaged

### API methods or options that are only available when running in Firefox

This library does not provide any polyfill for API methods and options that are only available on Firefox, and they are actually considered out of the scope of this library.

### tabs.executeScript

On Firefox `browser.tabs.executeScript` returns a promise which resolves to the result of the content script code that has been executed, which can be an immediate value or a Promise.

On Chrome, the `browser.tabs.executeScript` API method as polyfilled by this library also returns a promise which resolves to the result of the content script code, but only immediate values are supported.
If the content script code result is a Promise, the promise returned by `browser.tabs.executeScript` will be resolved to `undefined`.

### MSEdge support

MSEdge versions >= 79.0.309 are unofficially supported as a Chrome-compatible target (as for Opera or other Chrome-based browsers that also support extensions).

MSEdge versions older than 79.0.309 are **unsupported**, for extension developers that still have to work on extensions for older MSEdge versions, the MSEdge `--ms-preload` manifest key and the [Microsoft Edge Extension Toolkit](https://docs.microsoft.com/en-us/microsoft-edge/extensions/guides/porting-chrome-extensions)'s Chrome API bridge can be used to be able to load the webextension-polyfill without any MSEdge specific changes.

The following Github repository provides some additional detail about this strategy and a minimal test extension that shows how to put it together:

- https://github.com/rpl/example-msedge-extension-with-webextension-polyfill

## Contributing to this project

Read the [contributing section](CONTRIBUTING.md) for additional information about how to build the library from this repository and how to contribute and test changes.

[PR-114]: https://github.com/mozilla/webextension-polyfill/pull/114
[I-102]: https://github.com/mozilla/webextension-polyfill/issues/102#issuecomment-379365343
