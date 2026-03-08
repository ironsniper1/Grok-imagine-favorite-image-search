# Grok Imagine Favorites Search

A Tampermonkey userscript that adds a **full-text search interface** to your saved Grok Imagine images — bypassing Grok's virtual scroll to search your entire library, not just what's currently on screen.

---

## Features

- 🔍 **Search by prompt** — instantly filter your entire saved library by any keyword or phrase
- ⚡ **Persistent IndexedDB index** — thousands of images load in under a second after the first index
- 🔄 **Incremental updates** — on each visit, only fetches images you've liked since the last run
- 🚀 **Fast indexing** — first-time index runs as fast as the API allows with no artificial delays, fetching 40 images per request
- 📄 **Paginated results** — 20 results per page with Prev / Next / First / Last navigation
- 🗓 **Sort by Newest or Oldest**
- 🖼 **Own results grid** — renders matching images directly, completely bypassing Grok's virtualised scroll
- 💬 **Prompt tooltip** — hover any result card to see the full prompt
- 🖱 **Click to open** — click any image to open the full-resolution version in a new tab
- ⌨️ **Keyboard shortcuts** — `Ctrl/Cmd+F` to focus search, `←` `→` to page through results

---

## Installation

### Step 1 — Install Tampermonkey

Install the [Tampermonkey](https://www.tampermonkey.net/) browser extension:

- [Chrome](https://chrome.google.com/webstore/detail/tampermonkey/dhdgffkkebhmkfjojejmpbldmpobfkfo)
- [Firefox](https://addons.mozilla.org/en-US/firefox/addon/tampermonkey/)
- [Edge](https://microsoftedge.microsoft.com/addons/detail/tampermonkey/iikmkjmpaadaobahmlepeloendndfphd)
- [Safari](https://apps.apple.com/app/tampermonkey/id1482490089)

---

### Step 2 — Configure Tampermonkey (Do This Before Installing the Script)

After installing Tampermonkey, you need to enable two settings or the script will not work.

**Allow User Scripts:**

1. Go to `chrome://extensions` in your address bar
2. Find Tampermonkey and click **Details**
3. Scroll down and turn on **Allow User Scripts**

This allows Tampermonkey to run scripts that have not been reviewed by Google. You should only enable this if you trust the script you are installing.

**Allow in Incognito** *(optional)*:

On the same Details page, also turn on **Allow in Incognito** if you want the script to work in private/incognito windows. Without this the script will only run in normal browser windows.

Once both settings are enabled, proceed to installation below.

---

### Step 3 — Install the Script

1. Click the Tampermonkey icon in your browser toolbar
2. Select **Create a new script**
3. Delete all existing content in the editor
4. Paste the contents of [`grok-favorites-search.user.js`](./grok-favorites-search.user.js)
5. Press `Ctrl+S` (or `Cmd+S`) to save
6. Navigate to [grok.com/imagine/saved](https://grok.com/imagine/saved)

---

## Usage

### First Run

On your first visit after installing, the script will index your entire library. A status indicator in the search bar shows progress:

```
indexing… 1,240
indexing… 2,480
...
6,112 indexed
```

This only happens once. Subsequent loads read from the local IndexedDB cache instantly.

> ⏳ **Heads up:** Indexing time depends on how many images you have. A few hundred takes seconds, but a large library can take significantly longer. The script has to paginate through the entire API one batch at a time. It's best to leave the tab open and let it run in the background — or just kick it off before you go to sleep and it'll be ready in the morning. You only ever have to do this once.

> ⚠️ **API limit:** Grok's API caps how far back the liked feed goes in a single indexing session at around 6,000–6,100 posts. This is a hard server-side limit — not a bug in the script. Your index will naturally grow over time as the incremental update system picks up new likes on each visit, eventually building up a larger total index across multiple sessions.

### Searching

Type any word or phrase into the search bar. Multi-word searches require **all terms** to match (AND logic).

| Query | Matches |
|-------|---------|
| `dragon` | All prompts containing "dragon" |
| `red dragon forest` | Prompts containing all three words |
| `anime portrait` | Prompts containing both "anime" and "portrait" |

### Navigation

| Control | Action |
|---------|--------|
| `⏮ First` button | Jump to page 1 |
| `◀ Prev` button | Previous page |
| `Next ▶` button | Next page |
| `Last ⏭` button | Jump to last page |
| `←` / `→` arrow keys | Page through results (when search box is not focused) |
| `Ctrl/Cmd+F` | Focus the search bar |
| `Esc` | Unfocus the search bar |

### Sorting

Use the **Newest / Oldest** dropdown in the search bar to change the sort order of results. Sorting applies instantly without re-fetching.

### Incremental Updates

Every time you visit `grok.com/imagine`, the script:

1. Loads your full index from IndexedDB (instant)
2. Fetches the latest page from the API
3. Stops as soon as it hits an image it already knows
4. Saves any new images to the index

The status bar briefly shows `+N new (X total)` or `up to date`.

---

## How It Works

Grok's saved images page uses **virtualised rendering** — only the images currently visible in the viewport exist in the DOM. This makes it impossible to search by scanning the page directly.

This script works around that by:

1. **Calling the Grok API directly** (`/rest/media/post/list`) to fetch all your liked posts, including their prompt text
2. **Storing everything in IndexedDB** — a browser-native database with no meaningful size limit, persisted across sessions
3. **Rendering its own results grid** when you search, hiding Grok's native grid and displaying matching images from the index

---

## Storage

The script uses the browser's built-in **IndexedDB** to store your index locally. No data is sent anywhere — everything stays in your browser.

- **Database name:** `GrokSearchIndex`
- **Stored per post:** `id`, `prompt`, `thumbnail URL`, `media URL`, `createTime`
- **Typical size:** ~4–6 MB for 6,000+ posts

To clear the index, run this in your browser console on `grok.com/imagine/saved`:

```js
indexedDB.deleteDatabase('GrokSearchIndex');
```

Then refresh the page and the script will re-index from scratch.

---

## Checking Your Index

To verify how many posts are indexed, run this in the console on `grok.com/imagine/saved`:

```js
const req = indexedDB.open('GrokSearchIndex');
req.onsuccess = e => {
  const db = e.target.result;
  const tx = db.transaction('posts', 'readonly');
  const store = tx.objectStore('posts');
  const count = store.count();
  count.onsuccess = () => console.log(`✅ IndexedDB has ${count.result.toLocaleString()} posts indexed`);
  const sample = store.getAll(null, 3);
  sample.onsuccess = () => {
    console.log('Sample posts:');
    sample.result.forEach(p => console.log(`  • [${p.createTime?.slice(0,10)}] ${p.prompt?.slice(0,80)}`));
  };
};
```

---

## Compatibility

| Browser | Status |
|---------|--------|
| Chrome / Chromium | ✅ Tested |
| Firefox | ✅ Should work |
| Edge | ✅ Should work |
| Safari | ⚠️ Untested |

Requires Tampermonkey v4.0 or later.

---

## Known Limitations

- **API pagination cap** — Grok limits the liked feed to roughly 6,000–6,100 posts per indexing session. Older posts beyond this are not accessible in a single pass, but the index grows over time through incremental updates on each visit.
- **Posts with no prompt** — some posts (particularly auto-generated videos) have no prompt text. These are still indexed and counted but will not match any search term.
- **Unliked images** — removing a like removes it from future incremental updates but does not automatically remove it from the local index.
- **API changes** — if Grok changes their internal API, the script may need updating.

---

## Related Scripts

If you're also looking to **download** your Grok favorites in bulk, check out the companion script:

**[Grok Imagine – Bulk Favorites Downloader](./grok-bulk-downloader.user.js)**  
Downloads all your saved images in chunks, remembers what's already been downloaded, and supports folder-based organisation.

---

## License

MIT — do whatever you want with it.
