# X Keyword Cleaner

I used to shill for a project and later wanted to clean up the old posts/replies I made about it.

So I made this simple X keyword cleaner.

**Search your own X posts for a keyword → paste the script into your browser Console → it finds your matching posts/replies → you approve once → it deletes the matches.**

> **PC/Desktop only.** Best with Chrome, Brave, or Microsoft Edge.
>
> **Not affiliated with X.** X can change its website, so the script may need updates from time to time.

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

Click **Latest** and make sure the results are the posts/replies you actually want gone.

---

## 2. Open the Console

On that X search page:

**Right-click → Inspect → Console**

Windows shortcut:

```text
Ctrl + Shift + J
```

---

## 3. Put your keyword in the script

At the very top, find:

```javascript
const KEYWORDS = ['PUT YOUR KEYWORD HERE'];
```

Replace only the words inside the `[ ]`:

```javascript
const KEYWORDS = ['catwifdog'];
```

More than one spelling:

```javascript
const KEYWORDS = ['catwifdog', 'cat wif dog', '$catwifdog'];
```

**That's the only part you need to edit.**

---

## 4. Run it

1. Copy the whole script.
2. Paste it into the X Console.
3. Press **Enter**.
4. It scans your search first and shows what it found.
5. Click **OK** on the one popup if the results look right.
6. You will see X's normal **Delete** confirmation appear for each post. The script clicks it automatically.
7. Keep the X tab open while it runs.

The script only increases the deleted count after it verifies that the matching result disappeared. If X refuses a deletion, it retries instead of pretending it worked.

When it finishes, reload the same search and run it again if anything is still there.

---

## If it says `ERR_BLOCKED_BY_CLIENT`

That usually means an ad blocker/privacy extension blocked one of X's requests.

Temporarily pause the blocker for **x.com**, reload X, and run the cleaner again.

---

## Important

- Use it on **your own X account only**.
- You do **not** need to give anyone your X password, cookies, auth token, or API key.
- Deleting posts is permanent.
- The script only targets **your own posts/replies whose visible text contains the keyword you entered**.
- If X or your browser crashes, reopen the same search and run it again.

Emergency stop:

```javascript
window.X_KEYWORD_CLEANER_STOP = true
```

---

# Full script

You can also open [`keyword-wipe.js`](./keyword-wipe.js).

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
    LOAD_DELAY_MS: 2400,
    ERROR_BACKOFF_MS: 15000,
    COOLDOWN_EVERY: 20,
    COOLDOWN_MS: 10000,
    ACTION_RETRIES: 4,
    STALLS_TO_END_PASS: 6,
    MAX_DELETE_PASSES: 5,
    SHOW_CONFIRM_MS: 550,
    VERIFY_TIMEOUT_MS: 7000,
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

  function getOwnTweetId(article) {
    const target = username.toLowerCase();

    for (const link of article.querySelectorAll('a[href*="/status/"]')) {
      const href = link.getAttribute('href') || '';
      const match = href.match(/^\/([^/]+)\/status\/(\d+)/i);

      if (
        match &&
        match[1].toLowerCase() === target &&
        link.querySelector('time')
      ) {
        return match[2];
      }
    }

    return null;
  }

  function isOwnPost(article) {
    return Boolean(getOwnTweetId(article));
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

  function findOwnArticleById(id) {
    return visibleArticles().find((article) => getOwnTweetId(article) === id) || null;
  }

  function signature() {
    return visibleArticles()
      .map((article) => getOwnTweetId(article) || '?')
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

  async function verifyRemoved(id) {
    const start = Date.now();
    let missingChecks = 0;

    while (Date.now() - start < SETTINGS.VERIFY_TIMEOUT_MS) {
      if (window.X_KEYWORD_CLEANER_STOP) {
        throw new Error('Stopped.');
      }

      if (hasXError()) return 'error';

      if (!findOwnArticleById(id)) {
        missingChecks += 1;

        // Require several consecutive checks so a quick React rerender is
        // not mistaken for a successful deletion.
        if (missingChecks >= 3) return 'removed';
      } else {
        missingChecks = 0;
      }

      await sleep(350);
    }

    return 'still-there';
  }

  async function scrollSearchPass(onMatch) {
    window.scrollTo({ top: 0, behavior: 'auto' });
    await sleep(1600);

    let stalls = 0;
    let lastSignature = '';

    while (!window.X_KEYWORD_CLEANER_STOP && stalls < SETTINGS.STALLS_TO_END_PASS) {
      const articles = visibleArticles();
      let restartScan = false;

      for (const article of articles) {
        if (!article.isConnected || !matchesKeyword(article)) continue;

        const shouldRestart = await onMatch(article);

        // Important: after a delete, X rerenders the timeline. Throw away
        // the old article list and rescan instead of counting stale nodes.
        if (shouldRestart) {
          restartScan = true;
          break;
        }
      }

      if (restartScan) {
        stalls = 0;
        await sleep(700);
        continue;
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
    const id = getOwnTweetId(article);
    const originalText = getOwnPostText(article);

    if (!id || !article.isConnected || !matchesKeyword(article)) return false;

    for (let attempt = 1; attempt <= SETTINGS.ACTION_RETRIES; attempt += 1) {
      const currentArticle = findOwnArticleById(id);

      // A stale/disconnected article is NOT counted as a deletion.
      if (!currentArticle) {
        warn(`Lost sight of ${id} before deleting it. Rescanning instead of counting it.`);
        return false;
      }

      currentArticle.scrollIntoView({ block: 'center', behavior: 'auto' });
      await sleep(450);

      const caret = currentArticle.querySelector('[data-testid="caret"]');
      if (!caret) {
        warn(`Could not open the menu for ${id}. Retry ${attempt}/${SETTINGS.ACTION_RETRIES}`);
        await sleep(700);
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
        warn(`Could not find Delete for ${id}. Retry ${attempt}/${SETTINGS.ACTION_RETRIES}`);
        await sleep(850);
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
        warn(`Could not find the delete confirmation for ${id}. Retry ${attempt}/${SETTINGS.ACTION_RETRIES}`);
        await sleep(900);
        continue;
      }

      // Leave X's own Delete confirmation visible briefly. This makes the
      // action obvious to the user and gives the UI time to settle.
      await sleep(SETTINGS.SHOW_CONFIRM_MS);
      confirm.click();

      const result = await verifyRemoved(id);

      if (result === 'removed') {
        log(`VERIFIED DELETED ${id}: ${originalText.slice(0, 90)}`);
        return true;
      }

      if (result === 'error') {
        warn(`X reported an error for ${id}. Waiting before retrying...`);
        await sleep(SETTINGS.ERROR_BACKOFF_MS);
        continue;
      }

      warn(`Delete for ${id} was NOT verified. The post is still visible. Retry ${attempt}/${SETTINGS.ACTION_RETRIES}`);
      document.body.click();
      await sleep(1200);
    }

    warn(`Could not verify deletion of ${id}. It was NOT added to the deleted count.`);
    return false;
  }

  // Preview matching posts first.
  const previewIds = new Set();
  const previewRows = [];

  log(`Account: @${username}`);
  log(`Search: ${searchQuery}`);
  log(`Keyword(s): ${KEYWORDS.join(', ')}`);
  log('Scanning first. Nothing is being deleted yet.');

  await scrollSearchPass(async (article) => {
    const id = getOwnTweetId(article);
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
  console.log(`Verified deleted this run: ${deleted}`);
  console.log('Reload the same X search and check what remains. Rerun if needed.');
  console.log('==============================');
})();
```
