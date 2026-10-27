# Fall Studio

Two small black-and-white web toys about letting things fall apart, and putting them back.

- **Letterfall**: type a paragraph, press Collapse, and every letter falls like a building coming down and piles up on the floor.
- **Image Scrap**: upload a picture, press Scrap, and it tears into hundreds of pieces that drift down like leaves. Press Restore and the pieces fly back, but never fuse: thin white seams stay between them.
- **Welcome page**: a front door linking both tools.

It started as one sentence ("a landing page with a space to write, and a Collapse button, and all the text falls like a building") and grew through about a dozen rounds of "can you also…". This file records what exists, how it works, and how we got here.

---

## 1. Quick facts

| | |
|---|---|
| Author | Namdev, built in conversation with Claude (Sonnet 5.5) |
| Date | 26 to 27 October 2026 |
| Stack | Plain HTML, CSS and JavaScript. No framework, no build step |
| Libraries | Matter.js 0.19.0 (Letterfall only), loaded from cdnjs |
| Fonts | Newsreader (weight 300), IBM Plex Mono (300, 400), Noto Serif Devanagari (300), from Google Fonts |
| Hosting | Published as one multi-page Claude Artifact (private until shared) |
| Backend | None. Everything runs in the browser. Uploaded images never leave the device |
| Credentials | None. No API keys, accounts, tokens or secrets are used or stored anywhere (see section 9) |
| Theme | Pure black and white, light font weights only, no bold |

Published artifacts (private to the owner until shared from the Share menu):

- Welcome site (the current one, with Letterfall and Image Scrap inside): https://claude.ai/artifact/NkN11wTt49oH3cVbZcR8PR
- Older standalone Letterfall page: https://claude.ai/artifact/TQSA2AgX98Xb6nJHXJndmm
- Older standalone Image Scrap page: https://claude.ai/artifact/NDwz9EJpwuSUCnARnDteJt

The first two standalone links are earlier versions. The welcome site holds the latest code for both tools.

---

## 2. Site structure

The welcome site is a multi-file artifact. Three HTML pages are published together and link to each other with relative links.

```
index.html        Welcome page (headline animation + two cards)
letterfall.html   Letterfall tool
imagescrap.html   Image Scrap tool
```

- Each tool has a `← Welcome` link at the top left that goes to `index.html`.
- The welcome page cards link to `letterfall.html` and `imagescrap.html`.
- Other websites cannot be embedded inside a published artifact (blocked by the artifact security policy), so the tools are linked rather than iframed.
- `index.html` is published as the main page and gets the platform's page wrapper automatically. The two extra pages are served as-is, so they carry their own `<!doctype html>`, charset, viewport meta and a base style block. A small build script wrapped them.

Design rules shared by all three pages:

- Colors are only `#fff` and `#000`. Fonts use weight 300 for text and 300/400 for the small uppercase mono labels. No bold anywhere.
- Buttons are square, one-pixel black borders. Disabled buttons turn white with a dashed border.
- A horizontal "floor line" sits above the bottom bar. Letters and pieces rest on it.
- Layout works at phone width (about 400 px). Side gutters use `max(16px, env(safe-area-inset-*))`.
- Motion is skipped or reduced when the system asks for `prefers-reduced-motion`.

---

## 3. Letterfall: how it works

### What you see

1. A white page with "Write here." at the top center. Text you type is centered.
2. Bottom bar: `Collapse ↓`, `Reverse ↑`, `Sweep away`, and a letter count.
3. Press Collapse (or Ctrl/Cmd + Enter). The page trembles, then the bottom lines break first and the rest follow. Each letter is its own falling object.
4. Letters pile up on the floor line. Move the cursor (or a finger) through the pile and the letters are pushed like by a broom, so the text can be moved into any corner.
5. Reverse puts the last collapsed text back where it was. Sweep away opens the floor from left to right and drops the whole pile off screen.

### Technical details

- **Physics:** Matter.js 0.19.0. `engine.gravity.y = 1.8`, 8 position iterations, 5 velocity iterations, sleeping enabled so a settled pile costs almost nothing.
- **Fixed timestep:** the simulation advances in 1000/120 ms steps, at most 4 steps per frame, so behavior does not depend on the display refresh rate.
- **Speed cap:** `MAXV = 11` stops letters tunneling through each other or the floor.
- **Measuring the text:** the editor is a `contenteditable` div. On Collapse, a `TreeWalker` visits each text node and a `Range` measures every character's bounding box with `getBoundingClientRect()`. Characters are split with `Intl.Segmenter` (grapheme clusters) when available, so Marathi/Devanagari conjuncts and emoji stay whole. Fallback is `Array.from`.
- **One element per letter:** each character becomes an absolutely positioned `<span>` in an overlay layer, sized to its measured box, and moved with `transform: translate() rotate()`. Each letter also gets a Matter rectangle body (about 0.9 × width, 0.7 × height, with slightly rounded corners).
- **Cap:** at most 2000 letters per collapse (`MAX_LETTERS`).
- **Building collapse:** release time per letter is based on its vertical position: lowest line first, roughly 1100 ms across the visible text, plus a left-to-right offset (up to about 260 ms) and a random jitter (up to 180 ms). Before release there is a 450 ms "pre" phase with growing jitter and a screen shake.
- **Text below the visible area:** letters scrolled out of view are dropped in from above after about 1.2 to 2.1 seconds.
- **Screen shake:** collisions above a speed threshold add to a shake value that decays each frame. Skipped under reduced motion.
- **Reverse:** the letters' resting positions are removed from physics and animated back to re-measured target positions along curved paths with an ease-in-out cubic, starting from the top line, with a small random delay each. The original text is put back in the editor (hidden while measuring) and the flying spans are removed at the end.
- **Cursor brush:** pointer events on the whole stage. For each pointer move, the path is split into steps no longer than half the brush radius (64 px, 44 px on phones). Letters inside the radius get velocity from the pointer speed, a radial push and a slight lift, plus a bit of random spin. The invisible circle brush is intentionally not drawn.
- **Sweep away:** the floor is built from 24 static segments. They are removed from left to right over 1.4 to 2.8 seconds (scaled by screen width). The visible floor line shrinks with the same timing (`scaleX`, pivot on the right) so the line "disappears" as the floor opens. Sleeping letters above removed segments are woken. When everything has fallen off screen, the segments return and the line redraws left to right.
- **Paste:** pasted text is inserted as plain text only.

---

## 4. Image Scrap: how it works

### What you see

1. A dashed upload box ("Drop an image here."). Choose a file, drag and drop one, or paste an image.
2. The picture appears at the top center of the page, scaled to fit.
3. Bottom bar: `Scrap ↓`, `Restore ↑`, `Sweep away`, and a piece count.
4. Scrap: the picture cracks into about 840 irregular pieces (about 460 on phones). The pieces peel off **from the bottom first**, and the release travels up the picture like a waterfall. Each piece falls like a leaf.
5. Restore: pieces rise out of the pile and fly back to their original places along curved paths. They land with a thin white gap around each one, so the mended picture always shows crack lines.
6. A mended picture can be scrapped again. Sweep away clears the pile. Change image loads another picture.

### Upload rules

- Limit: **10 MB**. Larger files show a message with the actual size.
- Accepts anything the browser can decode as an image: PNG, JPG, WebP, GIF (first frame), BMP, AVIF, SVG and more.
- HEIC/HEIF photos usually cannot be decoded by browsers, so the page asks for a JPG instead.
- Loading uses an object URL on a normal `Image`, so nothing is uploaded anywhere.

### Technical details

- **Rendering:** one `<canvas>` over the page, 2D context, device pixel ratio capped at 2. The picture is drawn once into an off-screen "base" canvas at display size × DPR.
- **Cutting the picture (Voronoi shards):**
  - A jittered grid of seed points is made (jitter up to 40% of a cell). Target piece count is 840 on desktop and 460 on phones.
  - Each seed's cell starts as the whole image rectangle and is clipped by the perpendicular bisector (half-plane) of each nearby seed, up to 3 cells away in each direction. The clipping is Sutherland–Hodgman style.
  - Each cell polygon is then **shrunk inwards by a gap of 1.1 px** on every inner edge. Edges on the picture border are not shrunk, so the outer border stays crisp. The shrink is what creates the permanent white seams on restore.
  - Each shard gets its own small canvas sprite: the matching part of the picture is copied in, masked with the polygon using `destination-in`, then outlined with a 1.2 px white stroke to hide anti-aliasing fringes.
  - Centroid, area and a collision radius (about 0.62 × the equal-area circle radius) are computed per shard.
- **Release order (waterfall):** each shard's release time is `PRE (520 ms) + ((image bottom − shard y) / image height) × 2600 ms + (x position) × 260 ms + random up to 380 ms`. Bottom shards go first, the top last.
- **Leaf fall (not physics):** while airborne, a shard follows a hand-written curve so the motion stays smooth and predictable:
  - vertical speed eases toward a terminal speed of 120 to 210 px/s, with a gentle periodic speed wobble;
  - horizontal position follows a sine sway (amplitude 28 to 80 px, frequency about 1.0 to 2.3 rad/s) plus a slow drift;
  - rotation follows a cosine rock plus a slow spin;
  - a horizontal scale between about 0.18 and 1 simulates the leaf flipping edge-on.
  The sway ramps up over the first 1.2 seconds so pieces peel away gently.
- **Landing and the pile:** an airborne shard lands when it touches the floor or the circle of a resting shard. After landing it joins a small custom physics system:
  - circle colliders, a uniform grid spatial hash (cell size about twice the largest radius), 2 sub-steps per frame and 2 solver iterations;
  - gravity 1500 px/s², position correction, a very small bounce, friction;
  - side walls keep pieces on screen; the floor is analytic (a y value), with an `openEdge` x position that moves during Sweep so the floor is "open" to its left.
- **Sleeping:** a piece that moves slower than 110 px/s for 0.3 s falls asleep and is skipped by the solver. Slow pieces get extra damping so a pile settles in about two seconds. Pieces are woken by cursor brushing, hard impacts, or the sweep. When every piece is asleep, the page stops redrawing, so a resting pile uses almost no CPU/GPU.
- **Draw order:** shards are sorted by a z value each frame (landing order; airborne and restoring pieces always on top). Each shard is drawn with a single `setTransform` + `drawImage`.
- **Restore:** shards are sorted bottom-to-top by their current y, each starts after a delay (up to about 1 s plus jitter), flies along a curved path with an ease-in-out cubic over 1.2 to 1.7 s, and un-rotates and un-flips on the way. At the end the page switches to the "mended" state: the base picture stays hidden and the gapped shards themselves are drawn at home, so the seams are real.
- **Sweep:** the floor line shrinks left to right (same CSS technique as Letterfall). `openEdge` moves right at the same speed. Pieces over the open part wake up and fall. After the edge has passed, any piece still holding on gets woken and pushed down, so none can stay behind. Pieces that fall below the screen are removed.
- **Resize:** the picture and shards are rebuilt (when not animating); a pile is clamped to the new width.

---

## 5. Welcome page: how it works

- Headline "Let it fall." is split into one `<span>` per character. Every few seconds the letters drop a short distance into a reserved space under the headline, bounce slightly, rest, then lift back. Done with the Web Animations API (`element.animate`), with per-letter random rotation, offset and delay.
- Two cards, each with a small CSS-only looping preview: letters of "collapse" dropping onto a line, and eight striped/solid tiles drifting down like leaves.
- Cards are large links. Hover/focus inverts the "Open" button (black ↔ white).
- Footer line: "Black and white. Nothing is uploaded anywhere."
- Stacks to one column below 760 px.

---

## 6. How the work evolved (iteration log)

1. **First request:** a landing page with a writing area and a Collapse button; text falls letter by letter like a building and rests at the bottom. White theme, no bold, strong animation. Built Letterfall with Matter.js and one span per letter.
2. **Reverse + centered placeholder:** added a Reverse button that flies letters back; "Write here" moved to the center.
3. **Top placeholder + cursor brush:** "Write here" moved to the top center. Added the swiper-like cursor brush so the pile can be pushed into any corner.
4. **Invisible circle + sweep animation:** the visible brush ring was removed. First attempt at a sweep animation used a moving vertical line and an underline wipe on the Sweep button.
5. **Sweep redesigned:** the vertical line was rejected. The existing horizontal floor line now disappears left to right while the floor opens and the pile falls through, then redraws.
6. **"Image Scrap" idea:** upload an image (10 MB limit, all image types), Scrap and Restore buttons that mirror Collapse and Reverse, pieces falling like leaves, and permanent seams after restore. Discussed the plan first; agreed on: white gaps, a separate page.
7. **Image Scrap built:** Voronoi shards, analytic leaf flight, custom circle physics, white-gap restore. Tested in a headless browser with a generated test image.
8. **Welcome page + 50% smaller pieces + sweep bug:** added the welcome page linking both tools; piece size halved (about 220 to about 840 pieces); fixed the "half the pile stays" sweep bug (see section 7).
9. **Waterfall order + softer leaf fall:** pieces now release from the bottom of the image upward; leaf motion made slower, swayier and tippier.
10. **This README.**

---

## 7. Bugs found and how they were fixed

| Problem | Cause | Fix |
|---|---|---|
| Sweep left half the image pieces behind | A woken piece kept its old sleep timer. With very little speed right after waking, it fell asleep again immediately, hanging in mid-pile | Reset the sleep timer when waking; disable sleeping during Sweep; give the pile a final downward push once the floor is fully open |
| Resting pile ran at about 24 fps and never stopped computing | An overlap check woke neighbors on any deep overlap, which is constant in a packed pile, so nothing ever fell asleep | Wake neighbors only on a real impact (closing speed above 70 px/s); extra damping for slow pieces; raised the sleep speed threshold and shortened the sleep time. Resting pile now sleeps in about 2 s and runs at 60 fps in the test browser |
| Collision step ran for every piece, asleep or not | Loop did not skip sleeping pieces | Only awake pieces iterate; pairs with a sleeping piece are still handled |
| Upload box stayed visible after a picture loaded | The box's `display: flex` beat the `hidden` attribute in local testing | Added an explicit `[hidden] { display: none !important }` for those elements |
| Welcome headline letters dropped through the paragraph | Drop distance was based on the whole hero height | Reserved space under the headline and limited the drop to about a third of the font size |
| Sweep line rejected on Letterfall | A separate vertical line looked wrong | Use the existing floor line as the thing that disappears |
| Build script failed on `%` in CSS | Python `%` string formatting clashed with CSS percentages | Switched to plain `.replace()` placeholders |

---

## 8. Testing and performance

All testing was done with Playwright (headless Chromium, no GPU), using a generated 600 × 400 test picture and generated screenshots read back during development.

- Script syntax checked with `node --check` after every change.
- Flow tested end to end: upload → scrap → settle → brush → restore → scrap again → sweep → empty state, plus navigation Welcome → Image Scrap → Welcome.
- Phone width (400 px) screenshots checked for the welcome page and Image Scrap.
- Measured in the headless browser (software rendering): about 30 fps during the fall with 864 pieces, 60 fps once settled. Real devices with a GPU should be smoother. A very large photo on a slow phone may feel heavier.
- Not tested: real touch input, Safari/Firefox, real viewing inside the published artifact frame for the page-to-page links, Marathi text in Letterfall on a real device.

---

## 9. Credentials, privacy and credits

- **Credentials:** none. There are no API keys, passwords, tokens or accounts in any file. The pages call no APIs and have no server.
- **Privacy:** images are read locally with an object URL and drawn to a canvas. Nothing is sent over the network. Typed text stays on the page.
- **Network requests the pages do make:** Google Fonts (CSS and font files) and, for Letterfall, Matter.js from cdnjs. If these are blocked, the pages fall back to Georgia/system fonts, and Letterfall's physics would not run without Matter.js.
- **Sharing:** published artifacts are private to the owner until shared from the artifact's Share menu.
- **Credits:**
  - Matter.js (Liam Brummitt), MIT license: https://brm.io/matter-js/
  - Newsreader (Production Type), IBM Plex Mono (IBM) and Noto Serif Devanagari (Google), all under the SIL Open Font License, via Google Fonts.
  - Built by Namdev with Claude.

---

## 10. Known limitations and ideas for later

- Letterfall keeps only the most recent collapse for Reverse. A Sweep or a new collapse clears it.
- Letterfall is capped at 2000 letters per collapse.
- Image Scrap shows one picture at a time. GIFs and SVGs are treated as still images.
- HEIC photos are not supported by browsers; convert to JPG first.
- Pieces in Image Scrap are random each time; there is no "same cracks again" option.
- Ideas: a slower "cinematic" speed toggle; optional sound for impacts; downloading the mended picture with its seams as a PNG; choosing seam style (white or black) or piece count; letting Letterfall collapse the whole page; a piece-size slider.

---

## 11. Rebuilding from scratch

The source of all three pages is the published artifact itself. To get it back, open the welcome-site artifact link above and read the published files (`index.html`, `letterfall.html`, `imagescrap.html`). They are self-contained: no build step, no dependencies other than the two CDN items in section 9.

To host elsewhere, serve the three files from the same folder over any static host. Keep the file names, because the pages link to each other with relative paths (`index.html`, `letterfall.html`, `imagescrap.html`).
