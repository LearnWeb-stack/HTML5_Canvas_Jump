# HTML5_Canvas_Jump
Canvas_Jump
# Jump Game Tutorial: Collision, Jump Physics & Key Management

## 1. How Jump Physics Work

A jump is just two numbers changing every frame: `y` (height above ground) and `velY` (vertical speed). Gravity pulls `velY` down every frame; when you jump, you set `velY` to a large negative number so the player shoots upward.

```js
let y         = 0;       // height above ground in pixels
let velY      = 0;       // vertical speed (negative = moving up)
let hasJumped = false;   // lock — prevents jumping mid-air

const GRAVITY    = 0.6;  // added to velY every frame
const JUMP_FORCE = -12;  // negative because up = bottom increases
```

Every frame the loop does three things in order:

1. Add gravity to `velY` — this accelerates the fall
2. Subtract `velY` from `y` — moves the player up or down
3. If `y` hits zero the player has landed — reset everything

```js
velY += GRAVITY;   // accelerate downward
y    -= velY;      // move player (negative velY = moving up)
if (y <= 0) {
  y         = 0;
  velY      = 0;
  hasJumped = false;   // player is on the ground again
}

``` The math behind it:
At 60fps, GRAVITY = 0.6 means the player accelerates downward by 0.6px per frame, every frame. A full jump arc looks like this:

Frame 0: velY = -12 → player shoots up 12px
Each frame: velY += 0.6 → upward speed bleeds off
Frame 20: velY ≈ 0 → peak of the jump
Frame 40: velY = +12 → back on the ground

So the jump takes roughly 40 frames = 0.67 seconds at 60fps. That's a natural, snappy jump.
What happens if you change them:
GRAVITYJUMP_FORCEFeel0.6-12Default — snappy, arcade-like0.3-12Floaty — player hangs in the air1.2-12Heavy — fast fall, low arc0.6-18High jump, same fall speed0.6-6Short hop
The rule of thumb: the ratio JUMP_FORCE / GRAVITY controls jump height. -12 / 0.6 = 20, so the player rises for 20 frames before gravity wins. Double the ratio → double the height.
The values aren't special — you tune them until the jump feels right for your game. Platformers like Mario use a trick where gravity is stronger when the button is released mid-jump (variable jump height), but for a basic loop 0.6 and -12 are a solid starting point.
### Why subtract instead of add?

`y` is measured from the ground upward. When the player jumps, `velY` starts at `-12`. Subtracting a negative number increases `y`, so the player rises. As gravity adds `+0.6` each frame, `velY` climbs toward zero (peak) then goes positive (falling). Subtracting a positive `velY` decreases `y` — the player falls back down.
```
---

## 2. The hasJumped Lock

`hasJumped` is a one-shot boolean that prevents jumping while already in the air. Without it, holding Space would fire `JUMP_FORCE` every frame and the player would fly off screen.

```js
// When the player jumps:
velY      = JUMP_FORCE;
hasJumped = true;        // locked — no more jumps until landing

// When the player lands (y <= 0):
hasJumped = false;       // unlocked — ready to jump again
```

The lock is set on jump and cleared on landing. Nothing else touches it.

---

## 3. The Complete Game Loop

```js
function loop() {
  // 1. Horizontal movement
  x += velX;
  if (x >= maxX || x <= 0) velX *= -1;   // bounce off walls

  // 2. Gravity and vertical movement
  velY += GRAVITY;
  y    -= velY;
  if (y <= 0) { y = 0; velY = 0; hasJumped = false; }

  // 3. Apply position to the DOM element
  player.style.left   = x + 'px';
  player.style.bottom = (groundOffset + y) + 'px';

  requestAnimationFrame(loop);   // ~60 fps
}
```

`requestAnimationFrame` calls `loop` before the next screen paint — roughly 60 times per second. Each call moves the player one step forward in time.

---

## 4. Managing Keyboard Input

### How keydown works

`keydown` fires once when a key is first pressed, then repeatedly if held. For jumping you only care about the first press, which is why `hasJumped` does the filtering — not the event itself.

```js
document.addEventListener('keydown', e => {
  if (e.code === 'Space') {
    e.preventDefault();     // stop the page scrolling on Space
    if (!hasJumped) {
      velY      = JUMP_FORCE;
      hasJumped = true;
    }
  }
});
```

`e.preventDefault()` is important — without it, pressing Space scrolls the page down.

### e.code vs e.key

| Property | Returns | Example |
|----------|---------|---------|
| `e.code` | Physical key name | `"Space"`, `"ArrowUp"`, `"KeyW"` |
| `e.key`  | Character produced | `" "`, `"ArrowUp"`, `"w"` |

Use `e.code` for game controls — it is layout-independent and does not change if the player uses a different keyboard language.

### Common key codes

| Key | e.code |
|-----|--------|
| Spacebar | `Space` |
| Up Arrow | `ArrowUp` |
| Down Arrow | `ArrowDown` |
| Left Arrow | `ArrowLeft` |
| Right Arrow | `ArrowRight` |
| W | `KeyW` |
| A | `KeyA` |
| S | `KeyS` |
| D | `KeyD` |
| Enter | `Enter` |

### Supporting multiple jump keys

Extend the condition with `||` — any matching key triggers the same jump:

```js
document.addEventListener('keydown', e => {
  if (e.code === 'Space' || e.code === 'ArrowUp' || e.code === 'KeyW') {
    e.preventDefault();
    if (!hasJumped) {
      velY      = JUMP_FORCE;
      hasJumped = true;
    }
  }
});
```

### Tracking multiple keys at once

For movement in multiple directions (left/right while jumping), use a `keys` object instead of individual flags:

```js
const keys = {};

document.addEventListener('keydown', e => { keys[e.code] = true;  });
document.addEventListener('keyup',   e => { keys[e.code] = false; });

// Inside loop():
if (keys['ArrowLeft'])  x -= speed;
if (keys['ArrowRight']) x += speed;
if (keys['Space'] && !hasJumped) {
  velY      = JUMP_FORCE;
  hasJumped = true;
}
```

`keyup` clears the key when released, so holding left and right simultaneously is handled correctly.

---

## 5. Collision Detection

A collision check tests whether two rectangles overlap. Each rectangle needs a position (`x`, `y`) and a size (`width`, `height`).

```js
function isColliding(ax, ay, aw, ah,   // rect A: x, y, width, height
                     bx, by, bw, bh) { // rect B
  return ax < bx + bw &&   // A left edge is left of B right edge
         ax + aw > bx &&   // A right edge is right of B left edge
         ay < by + bh &&   // A bottom edge is below B top edge
         ay + ah > by;     // A top edge is above B bottom edge
}
```

All four conditions must be true at the same time for the rectangles to overlap. If any one is false, there is a gap between them.

### Using it in the loop

```js
const playerX = x;
const playerY = y;
const wallX   = 400;
const wallY   = 0;
const wallH   = 80;

if (isColliding(playerX, playerY, 30, 30, wallX, wallY, 20, wallH)) {
  // player is touching the wall — do something
}
```

### Collision without autojump

If you want to detect the wall but let the player choose when to jump, just use collision for visual feedback, a score, or a game-over condition — and keep jump on the keydown listener only:

```js
// In the loop — detection only, no forced jump
if (isColliding(playerX, playerY, 30, 30, wallX, wallY, 20, wallH)) {
  wall.style.background = '#ff4444';   // flash red on contact
} else {
  wall.style.background = '#f5a623';   // normal colour
}
```

---

## 6. Putting It All Together

```js
// Constants
const GRAVITY    = 0.6;
const JUMP_FORCE = -12;
const groundOffset = 40;

// State
let x = 50, velX = 2;
let y = 0,  velY = 0;
let hasJumped = false;
const maxX = arena.offsetWidth - 30;

// Keys
document.addEventListener('keydown', e => {
  if (e.code === 'Space' || e.code === 'ArrowUp') {
    e.preventDefault();
    if (!hasJumped) {
      velY      = JUMP_FORCE;
      hasJumped = true;
    }
  }
});

// Loop
function loop() {
  // Horizontal
  x += velX;
  if (x >= maxX || x <= 0) velX *= -1;

  // Vertical
  velY += GRAVITY;
  y    -= velY;
  if (y <= 0) { y = 0; velY = 0; hasJumped = false; }

  // Collision check (no autojump)
  if (isColliding(x, y, 30, 30, 400, 0, 20, 80)) {
    wall.style.background = '#ff4444';
  } else {
    wall.style.background = '#f5a623';
  }

  // DOM
  player.style.left   = x + 'px';
  player.style.bottom = (groundOffset + y) + 'px';

  requestAnimationFrame(loop);
}

requestAnimationFrame(loop);
```
