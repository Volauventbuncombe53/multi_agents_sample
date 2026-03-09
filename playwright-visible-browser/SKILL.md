---
name: playwright-visible-browser
description: Start a visible Chromium browser for `playwright-interactive` in this repo's Codex `js_repl` environment. Use when headed Playwright needs a reliable launch recipe, especially for `./multi_agents/start_agent finder`, where `await import("playwright")` and inherited display variables can fail.
---

# Playwright Visible Browser

Use this skill together with `playwright-interactive` when you need a real headed Chromium window that the user can watch.

## When To Use

Use this skill when:

- `./multi_agents/start_agent finder` needs a visible browser
- `playwright-interactive` fails before or during headed browser launch
- `await import("playwright")` is unreliable in `js_repl`
- Playwright reports `Missing X server or $DISPLAY`

In this repo, the verified headed launch recipe is:

- load Playwright with `createRequire(...)` from the repo `package.json`
- launch Chromium with `env: { ...process.env, DISPLAY: ':0' }`

Do not start by adding `XAUTHORITY` or other display overrides unless the simple `DISPLAY: ':0'` launch actually fails.

## Bootstrap Override

Run this instead of the default `playwright-interactive` import/bootstrap when you need visible Chromium:

```javascript
var chromium;
var browser;
var context;
var page;
var mobileContext;
var mobilePage;
var pwRequire;

{
  const { createRequire } = await import('node:module');
  pwRequire = createRequire(`${process.cwd()}/package.json`);
  ({ chromium } = pwRequire('playwright'));
}

var resetWebHandles = function () {
  context = undefined;
  page = undefined;
  mobileContext = undefined;
  mobilePage = undefined;
};

var ensureVisibleBrowser = async function () {
  if (browser && !browser.isConnected()) {
    browser = undefined;
    resetWebHandles();
  }

  browser ??= await chromium.launch({
    headless: false,
    env: { ...process.env, DISPLAY: ':0' },
  });

  return browser;
};
```

## Desktop Session

```javascript
var TARGET_URL = 'http://127.0.0.1:15373';

if (page?.isClosed()) page = undefined;

await ensureVisibleBrowser();
context ??= await browser.newContext({
  viewport: { width: 1600, height: 900 },
});
page ??= await context.newPage();

await page.goto(TARGET_URL, { waitUntil: 'domcontentloaded' });
console.log('Loaded:', await page.title());
```

## Native-Window Pass

```javascript
var TARGET_URL = 'http://127.0.0.1:15373';

await ensureVisibleBrowser();

await page?.close().catch(() => {});
await context?.close().catch(() => {});
page = undefined;
context = undefined;

context = await browser.newContext({ viewport: null });
page = await context.newPage();

await page.goto(TARGET_URL, { waitUntil: 'domcontentloaded' });
console.log('Loaded native window:', await page.title());
```

## Smoke Test

If you only need to prove the browser can open visibly before starting the real investigation:

```javascript
await ensureVisibleBrowser();
const probeContext = await browser.newContext({ viewport: { width: 1280, height: 800 } });
const probePage = await probeContext.newPage();
await probePage.goto('about:blank');
await probePage.waitForTimeout(1500);
await probeContext.close();
console.log('Visible browser launch ok');
```

## Notes

- Prefer `127.0.0.1` over `localhost` for local servers.
- Reuse the same `browser`, `context`, and `page` bindings across `js_repl` cells.
- If the browser handle goes stale, set `browser = undefined` and rerun the helper cell.
- In this repo, `await import('playwright')` is less reliable than `createRequire(...)` from the workspace `package.json`.

## Cleanup

When the investigation is finished:

```javascript
await page?.close().catch(() => {});
await context?.close().catch(() => {});
await mobilePage?.close().catch(() => {});
await mobileContext?.close().catch(() => {});
await browser?.close().catch(() => {});

page = undefined;
context = undefined;
mobilePage = undefined;
mobileContext = undefined;
browser = undefined;
```
