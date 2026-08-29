# TikTok Mass Unlike

> [!CAUTION]
> TikTok rate limits bulk unliking. If the console counter keeps climbing but your liked tab doesn't shrink, your requests are being silently dropped. Stop, wait an hour, and verify before running another batch. Automating actions on your account carries some risk of being flagged. Use at your own risk.

Removes liked videos from your TikTok account, one batch at a time.

Built after hitting 10k+ likes with no bulk-remove option anywhere in the app.

> [!NOTE]
> This only works on the web version at tiktok.com. The desktop and mobile apps have no console to paste into.

## How to use this script

1. Go to your profile and open the **Liked** tab
2. Click the first video so it opens in the player
3. Press `F12` to open DevTools
4. Go to the **Console** tab
5. Paste the following code and hit enter:

<details>
<summary>Click to expand</summary>

```javascript
/**
 * tiktok-mass-unlike
 * Removes liked videos from your TikTok account via the web UI.
 */

const CONFIG = {
  batchSize: 300,             // videos per run
  heartTimeout: 4000,         // ms to wait for the next video's heart
  clickDelay: 250,            // ms after clicking, lets the unlike register
  advanceDelay: 400,          // ms after pressing ArrowDown
  maxMisses: 5,               // consecutive failures before giving up
  likedColor: '255, 59, 92',  // TikTok red = video is currently liked
};

window.STOP = false;

const sleep = ms => new Promise(r => setTimeout(r, ms));

/**
 * TikTok keeps the liked grid mounted behind the player, so multiple hearts
 * exist in the DOM. Filter to on-screen elements and take the rightmost,
 * which is the player's action bar.
 */
function findHeart() {
  const all = [...document.querySelectorAll(
    '[class*="HeartWrapper"], [data-e2e="like-icon"], [data-e2e="browse-like-icon"]'
  )];

  const visible = all.filter(el => {
    const r = el.getBoundingClientRect();
    return r.width > 0 && r.height > 0 && r.top >= 0 && r.top < window.innerHeight;
  });

  visible.sort((a, b) => b.getBoundingClientRect().left - a.getBoundingClientRect().left);
  return visible[0] || null;
}

/** A red heart means the video is liked. Prevents accidentally re-liking. */
function isLiked(heart) {
  const svg = heart.querySelector('svg') || heart;
  return getComputedStyle(svg).color.includes(CONFIG.likedColor);
}

/** Poll for a red heart rather than sleeping a fixed amount. */
async function waitForRedHeart(maxMs = CONFIG.heartTimeout) {
  const start = Date.now();
  while (Date.now() - start < maxMs) {
    if (window.STOP) return null;
    const heart = findHeart();
    if (heart && isLiked(heart)) return heart;
    await sleep(100);
  }
  return null;
}

function nextVideo() {
  document.dispatchEvent(new KeyboardEvent('keydown', {
    key: 'ArrowDown', code: 'ArrowDown', keyCode: 40, which: 40, bubbles: true,
  }));
}

async function unlikeBatch(limit = CONFIG.batchSize) {
  let count = 0;
  let misses = 0;

  console.log(`starting, target ${limit}. type  STOP = true  to halt`);

  while (count < limit) {
    if (window.STOP) { console.log('stopped by user'); break; }

    const heart = await waitForRedHeart();

    if (heart) {
      misses = 0;
      (heart.closest('button') || heart).click();
      count++;
      console.log(`unliked ${count} / ${limit}`);
      await sleep(CONFIG.clickDelay);
    } else {
      if (window.STOP) break;
      misses++;
      console.log(`no red heart (${misses}/${CONFIG.maxMisses})`);
      if (misses >= CONFIG.maxMisses) {
        console.log('stopping: reached the end, or TikTok is rate limiting');
        break;
      }
    }

    nextVideo();
    await sleep(CONFIG.advanceDelay);
  }

  console.log(`batch done. unliked ${count}`);
  console.log('run  unlikeBatch()  again for another batch');
}

unlikeBatch();
```

</details>

(If you're unable to paste into the console, you might have to type `allow pasting` and hit enter)

6. Leave the tab visible and let it run. It stops after 300 videos
7. Run `unlikeBatch()` again for the next batch

You can track progress by watching the `unliked N / 300` prints in the Console tab.

To stop early:

```js
STOP = true
```

Or just refresh the page.

## How it works

The script finds the like button in the player, confirms the heart is red (meaning currently liked), clicks it, then presses ArrowDown to advance to the next video. It polls for the heart every 100ms rather than sleeping a fixed amount, so it moves as fast as the page loads.

## Config

| Option | Default | What it does |
|---|---|---|
| `batchSize` | `300` | Videos to unlike per run |
| `heartTimeout` | `4000` | Max ms to wait for the next heart before counting a miss |
| `clickDelay` | `250` | Pause after clicking so the unlike registers |
| `advanceDelay` | `400` | Pause after moving to the next video |
| `maxMisses` | `5` | Consecutive misses before stopping |

Raise `clickDelay` and `advanceDelay` if unlikes stop registering.

## FAQ

**Q: Why not just send requests to the API directly? That would be way faster**

A: TikTok's `webmssdk.js` signs every request with `X-Gnarly` and `X-Dynosaur` parameters computed over the full URL, including the video ID. A hand-written `fetch` to `/api/commit/item/digg/` gets rejected with `Web SDK blocked`. Clicking the real button lets TikTok's own SDK do the signing, which is why the UI approach works and direct requests don't.

**Q: Can I replay a captured request from the Network tab?**

A: It works for that exact video and then stops. Swap in a different `aweme_id` and the signature no longer matches.

**Q: It prints `no red heart` over and over**

A: A video needs to be open in the player, not the grid of thumbnails. Click into a liked video first. If a video is open and it still fails, TikTok changed their markup, inspect the heart and update the selectors in `findHeart()`.

**Q: The counter goes up but my likes are still there**

A: You're rate limited. Stop, wait an hour, then run a smaller batch with larger delays.

**Q: The same video repeats forever**

A: The page lost keyboard focus. Click once on the video area, then rerun.

**Q: Console shows `Promise {<pending>}`**

A: That's normal. The function is async so the console prints the pending promise immediately. The script is still running, look for the log lines underneath.

**Q: How long does 10k likes take?**

A: Roughly an hour per 1000 when the page keeps up, so several hours spread across sessions. Leave it in a side window.

**Q: Can it run in a background tab?**

A: No. Browsers throttle timers in hidden tabs. Keep the window visible, even if it's off to the side.

**Q: Is there a faster version?**

A: Not one that works. See the first FAQ entry.

## Disclaimer

Use on your own account. This automates clicks you could make manually. Provided as-is, with no warranty.
