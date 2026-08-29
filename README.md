# tiktok-mass-unlike

A browser console script that removes liked videos from your TikTok account.

Built out of necessity after accumulating 10k+ likes with no bulk-remove
option in the app.

## Usage

1. Go to your profile, open the **Liked** tab
2. Click the first video so it opens in the player
3. Open DevTools (`F12`) → **Console**
4. Paste `tiktok-unlike.js`, press Enter

It runs a batch of 300, then stops. Run `unlikeBatch()` again for the next batch.

To stop early:

```js
STOP = true
```

Or just refresh the page.

## How it works

The script finds the like button in the player, checks the heart is red
(meaning currently liked), clicks it, then presses ArrowDown to advance to
the next video.

## Why not just call the API?

TikTok's `webmssdk.js` signs every request with `X-Gnarly` and `X-Dynosaur`
parameters computed over the full URL. A hand-written `fetch` to
`/api/commit/item/digg/` gets rejected:
