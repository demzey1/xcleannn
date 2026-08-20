# X Keyword Cleaner

I used to shill for a project and later wanted to clean up the old posts/replies I made about it.

So I made this simple X keyword cleaner.

**Search your own X posts for a keyword → paste the script into your browser Console → it finds your matching posts/replies → you approve once → it deletes the matches.**

> **PC/Desktop only.** Best with Chrome, Brave, or Microsoft Edge.
>
> **Not affiliated with X.** X can change its website, so the script may need updates from time to time.

## Demo

**Demo video:** `PASTE_YOUR_DEMO_VIDEO_LINK_HERE`

---

# How to use

## 1. Search for the old posts/replies

On X, search:

```text
from:YOUR_USERNAME YOUR_KEYWORD
```

Example:

```text
from:demzey1 catwifdog
```

Then click **Latest** and make sure the results are the posts/replies you actually want gone.

---

## 2. Open the Console

On the X search page:

**Right-click → Inspect → Console**

Windows shortcut:

```text
Ctrl + Shift + J
```

Mac shortcut:

```text
Command + Option + J
```

---

## 3. Put your keyword in the script

At the very top of the script, find:

```javascript
const KEYWORDS = ['PUT YOUR KEYWORD HERE'];
```

Replace the text inside the `[ ]` with your keyword:

```javascript
const KEYWORDS = ['catwifdog'];
```

For more than one spelling/name:

```javascript
const KEYWORDS = ['catwifdog', 'cat wif dog', '$catwifdog'];
```

**That's the only part you need to edit.**

---

## 4. Run it

1. Copy the full script below.
2. Paste it into the X Console.
3. Press **Enter**.
4. It will scan and show what it found first.
5. A normal browser popup will ask if you want to delete the matching results.
6. Click **OK** to delete them, or **Cancel** to stop.
7. Keep the X tab open while it runs.

When it finishes, reload the same X search. If X missed a few or stopped loading, run it again.

---

## Important

- Use it on **your own X account only**.
- You do **not** need to give anyone your X password, cookies, auth token, or API key.
- Deleting posts is permanent.
- The script only deletes **your own posts/replies whose visible text contains the keyword you entered**.
- If your reply was something like `bullish` and did **not** actually contain the project name, this keyword cleaner may not find it.
- If the browser/X crashes, reopen the same search and run the script again.

To stop a running cleaner, reload/close the tab or enter:

```javascript
window.X_KEYWORD_CLEANER_STOP = true
```

---

# Full script

Open [`keyword-wipe.js`](./keyword-wipe.js) or copy it below.

```javascript
(async () => {
  'use strict';

  // ============================================================
  // X KEYWORD CLEANER
  // ONLY CHANGE THE WORD(S) INSIDE THE [ ] BELOW.
  // ============================================================

  const KEYWORDS = ['PUT YOUR KEYWORD HERE'];

  // ============================================================
  // DO NOT EDIT ANYTHING BELOW THIS LINE.
  // ============================================================

  const SETTINGS = {
    MAX_DELETIONS: 500,
    ACTION_DELAY_MS: 1700,
    LOAD_DELAY_MS: 2400,
    ERROR_BACKOFF_MS: 20000,
    COOLDOWN_EVERY: 25,
    COOLDOWN_MS: 12000,
    ACTION_RETRIES: 4,
    STALLS_TO_END_PASS: 6,
    MAX_DELETE_PASSES: 4,
  };

  const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));
  const cleanText = (value) => (value || '').replace(/\s+/g, ' ').trim();
  const log = (...args) => console.log('[X-KEYWORD-CLEANER]', ...args);
  const warn = (...args) => console.warn('[X-KEYWORD-CLEANER]', ...args);

  if (location.hostname !== 'x.com') {
    throw new Error('Open x.com first, then run the script again.');
  }

  if (!location.pathname.startsWith('/search')) {
    throw new Error('Open X Search first. Example: from:YOUR_USERNAME yourkeyword');
  }

  const keywords = KEYWORDS
    .map((keyword) => String(keyword).trim().toLowerCase())
    .filter(Boolean);

  if (
    keywords.length === 0 ||
    keywords.some((keyword) => keyword === 'put your keyword here')
  ) {
    throw new Error("Replace 'PUT YOUR KEYWORD HERE' with the word/project you want to clean first.");
  }

  window.X_KEYWORD_CLEANER_STOP = false;

  const profileHref = document
    .querySelector('a[data-testid="AppTabBar_Profile_Link"]')
    ?.getAttribute('href');

  const username = profileHref?.match(/^\/([^/?#]+)/)?.[1];

  if (!username) {
    throw new Error('Could not detect your X username. Open your profile once, then return to the search page.');
  }

  const searchQuery = new URL(location.href).searchParams.get('q') || '';
  const escapedUsername = username.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
  const fromPattern = new RegExp(`(^|\\s)from:${escapedUsername}(?=\\s|$)`, 'i');

  if (!fromPattern.test(searchQuery)) {
    throw new Error(`Safety check: your X search must include from:${username}`);
  }

  function getTweetId(article) {
    for (const link of article.querySelectorAll('a[href*="/status/"]')) {
      const match = (link.getAttribute('href') || '').match(/\/status\/(\d+)/);
      if (match) return match[1];
    }
    return null;
  }

  function isOwnPost(article) {
    const target = username.toLowerCase();

    return [...article.querySelectorAll('a[href*="/status/"]')].some((link) => {
      const href = link.getAttribute('href') || '';
      const match = href.match(/^\/([^/]+)\/status\/(\d+)/i);

      return Boolean(
        match &&
        match[1].toLowerCase() === target &&
        link.querySelector('time')
      );
    });
  }

  function getOwnPostText(article) {
    const textElement = article.querySelector('[data-testid="tweetText"]');
    return cleanText(textElement?.innerText || textElement?.textContent || '');
  }

  function matchesKeyword(article) {
    if (!isOwnPost(article)) return false;
    const text = getOwnPostText(article).toLowerCase();
    return keywords.some((keyword) => text.includes(keyword));
  }

  function visibleArticles() {
    return [...document.querySelectorAll('article[data-testid="tweet"]')]
      .filter((article) => article.isConnected);
  }

  function signature() {
    return visibleArticles()
      .map((article) => getTweetId(article) || '?')
      .join('|');
  }

  function atBottom() {
    const height = Math.max(
      document.body.scrollHeight,
      document.documentElement.scrollHeight
    );

    return window.scrollY + window.innerHeight >= height - 450;
  }

  async function waitFor(getter, timeout = 4000, interval = 90) {
    const start = Date.now();

    while (Date.now() - start < timeout) {
      if (window.X_KEYWORD_CLEANER_STOP) {
        throw new Error('Stopped.');
      }

      const result = getter();
      if (result) return result;
      await sleep(interval);
    }

    return null;
  }

  function hasXError() {
    const alerts = [
      ...document.querySelectorAll('[data-testid="toast"]'),
      ...document.querySelectorAll('[role="alert"]'),
    ];

    return alerts.some((element) =>
      /something went wrong|try again|rate limit|too many requests|error/i.test(
        cleanText(element.textContent)
      )
    );
  }

  async function scrollSearchPass(onMatch) {
    window.scrollTo({ top: 0, behavior: 'auto' });
    await sleep(1600);

    let stalls = 0;
    let lastSignature = '';

    while (!window.X_KEYWORD_CLEANER_STOP && stalls < SETTINGS.STALLS_TO_END_PASS) {
      const articles = visibleArticles();

      for (const article of articles) {
        if (!article.isConnected || !matchesKeyword(article)) continue;
        const shouldRestart = await onMatch(article);
        if (shouldRestart) {
          stalls = 0;
          await sleep(450);
          continue;
        }
      }

      const before = signature();
      const beforeY = window.scrollY;

      window.scrollBy({
        top: Math.max(800, Math.floor(window.innerHeight * 0.9)),
        behavior: 'auto',
      });

      await sleep(SETTINGS.LOAD_DELAY_MS);

      const after = signature();
      const afterY = window.scrollY;

      const stalled =
        after === before &&
        after === lastSignature &&
        Math.abs(afterY - beforeY) < 50 &&
        atBottom();

      if (stalled) {
        stalls += 1;
        log(`Checking for more results... ${stalls}/${SETTINGS.STALLS_TO_END_PASS}`);

        window.scrollBy({
          top: -Math.max(450, Math.floor(window.innerHeight * 0.5)),
          behavior: 'auto',
        });
        await sleep(650);

        window.scrollBy({
          top: Math.max(900, Math.floor(window.innerHeight * 1.05)),
          behavior: 'auto',
        });
        await sleep(SETTINGS.LOAD_DELAY_MS);
      } else {
        stalls = 0;
      }

      lastSignature = after;
    }
  }

  async function deleteArticle(article) {
    const id = getTweetId(article);
    if (!id || !article.isConnected || !matchesKeyword(article)) return false;

    for (let attempt = 1; attempt <= SETTINGS.ACTION_RETRIES; attempt += 1) {
      if (!article.isConnected) return true;

      article.scrollIntoView({ block: 'center', behavior: 'auto' });
      await sleep(450);

      const caret = article.querySelector('[data-testid="caret"]');
      if (!caret) {
        await sleep(650);
        continue;
      }

      caret.click();

      const deleteItem = await waitFor(() => {
        const menus = [...document.querySelectorAll('[role="menu"]')];
        const menu = menus[menus.length - 1] || document;

        return [...menu.querySelectorAll('[role="menuitem"]')].find((element) =>
          /^delete(?: post)?$/i.test(cleanText(element.textContent))
        );
      });

      if (!deleteItem) {
        document.body.click();
        warn(`Could not find Delete for ${id}. Retrying...`);
        await sleep(800);
        continue;
      }

      deleteItem.click();

      const confirm = await waitFor(() => {
        const byTestId = document.querySelector('[data-testid="confirmationSheetConfirm"]');
        if (byTestId) return byTestId;

        const dialog = document.querySelector('[role="dialog"]');
        if (!dialog) return null;

        return [...dialog.querySelectorAll('button')].find((button) =>
          /^delete$/i.test(cleanText(button.textContent))
        );
      });

      if (!confirm) {
        document.body.click();
        warn(`Could not find the delete confirmation for ${id}. Retrying...`);
        await sleep(900);
        continue;
      }

      confirm.click();
      await sleep(SETTINGS.ACTION_DELAY_MS);

      if (hasXError()) {
        warn('X returned an error. Waiting before trying again...');
        await sleep(SETTINGS.ERROR_BACKOFF_MS);
        continue;
      }

      log(`Deleted ${id}: ${getOwnPostText(article).slice(0, 90)}`);
      return true;
    }

    warn(`Skipped ${id} after repeated errors. You can rerun the script later.`);
    return false;
  }

  // Preview the matching posts first.
  const previewIds = new Set();
  const previewRows = [];

  log(`Account: @${username}`);
  log(`Search: ${searchQuery}`);
  log(`Keyword(s): ${KEYWORDS.join(', ')}`);
  log('Scanning first. Nothing is being deleted yet.');

  await scrollSearchPass(async (article) => {
    const id = getTweetId(article);
    if (!id || previewIds.has(id)) return false;

    previewIds.add(id);
    const text = getOwnPostText(article);
    previewRows.push({ id, text: text || '[No visible text]' });
    log(`FOUND ${id}: ${text || '[No visible text]'}`);
    return false;
  });

  console.table(previewRows);

  if (previewRows.length === 0) {
    log('No matching posts/replies were found in this X search.');
    return;
  }

  const approved = window.confirm(
    `X Keyword Cleaner found ${previewRows.length} matching post(s)/reply(ies) for: ${KEYWORDS.join(', ')}\n\n` +
    'Click OK to delete these matching results.\nClick Cancel to stop without deleting anything.'
  );

  if (!approved) {
    log('Cancelled. Nothing was deleted.');
    return;
  }

  let deleted = 0;

  for (let pass = 1; pass <= SETTINGS.MAX_DELETE_PASSES; pass += 1) {
    let deletedThisPass = 0;
    log(`Delete pass ${pass}/${SETTINGS.MAX_DELETE_PASSES}...`);

    await scrollSearchPass(async (article) => {
      if (deleted >= SETTINGS.MAX_DELETIONS) {
        window.X_KEYWORD_CLEANER_STOP = true;
        warn(`Stopped at the ${SETTINGS.MAX_DELETIONS}-post safety limit.`);
        return false;
      }

      if (await deleteArticle(article)) {
        deleted += 1;
        deletedThisPass += 1;

        if (deleted % SETTINGS.COOLDOWN_EVERY === 0) {
          log('Taking a short cooldown...');
          await sleep(SETTINGS.COOLDOWN_MS);
        }

        return true;
      }

      return false;
    });

    if (window.X_KEYWORD_CLEANER_STOP) break;
    if (deletedThisPass === 0) break;
  }

  console.log('');
  console.log('==============================');
  console.log('X KEYWORD CLEANER FINISHED');
  console.log(`Deleted this run: ${deleted}`);
  console.log('Reload the same X search and check what remains. Rerun if needed.');
  console.log('==============================');
})();
```

---

## If it stops working

X changes its website sometimes. Open an issue on this repo and say what happened.

**Never post your password, cookies, auth token, or private session information in an issue.**
