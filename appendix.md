# Appendix: formulas behind the bhop graphs

This appendix lists the closed-form formulas behind every graph in the [main document](README.md), and then walks through how to derive them from scratch.

Table of Contents:
- [Appendix: formulas behind the bhop graphs](#appendix-formulas-behind-the-bhop-graphs)
- [Notation](#notation)
- [CS:GO formulas](#csgo-formulas)
	- [Perfect bhop chance vs scroll speed](#perfect-bhop-chance-vs-scroll-speed)
	- [Jump cooldown](#jump-cooldown)
- [CS2 formulas](#cs2-formulas)
	- [Time spent in the last 2 units](#time-spent-in-the-last-2-units)
	- [Window size for one landing](#window-size-for-one-landing)
	- [Average window size vs fall speed](#average-window-size-vs-fall-speed)
	- [Perfect bhop chance](#perfect-bhop-chance)
	- [Jump cooldown in CS2](#jump-cooldown-in-cs2)
	- [Reference values](#reference-values)
- [How to derive the CS2 formulas yourself](#how-to-derive-the-cs2-formulas-yourself)

# Notation

All CS2 times are measured in **ticks** (1 tick = 15.625ms). Multiply by 15.625 to get milliseconds.

| Symbol | Meaning | Value |
|---|---|---|
| `g` | gravity | 800 u/s² |
| `v` | fall speed when reaching the floor | u/s |
| `w` | half of `sv_bhop_time_window` | 0.25 tick (3.9ms) |
| `d` | time spent falling through the last 2 units above the floor | ticks, see below |
| `a` | time from entering the last 2 units to the next tick end | uniform between 0 and 1 tick |
| `T` | time between two jump presses (scroll gap) | ticks, e.g. 16ms = 1.024 ticks |

`a` is what captures "where in the tick the landing happens". For a player landing from a given height, every value of `a` between 0 and 1 is equally likely, so every average below is an average over `a`.

# CS:GO formulas

## Perfect bhop chance vs scroll speed
Let `t` be the tick length (15.625ms on 64 tick, 7.8125ms on 128 tick) and `G` the scroll gap in ms. With `T = G / t`:

```
chance = 0              if T < 1
chance = 1 - 1/T        if 1 <= T <= 2
chance = 1/T            if T >= 2
```

A perfect bhop needs a press in the input packet of the first tick on the ground (probability `1/T` for `T >= 1`), and no press in the packet before it (probability `min(1, T - 1)` given the first). Multiplying gives `1 - 1/T` below 2 ticks and `1/T` above. The peak is 50% at `T = 2`.

For example, on 64 tick with a 24ms gap, T = 24 / 15.625 = 1.536, so the chance is 1 − 1/1.536 = 34.9%. With a 40ms gap, T = 2.56 and the chance is 1/2.56 = 39.1%. With exactly 31.25ms, T = 2 and the chance is 50%.

The CS:GO window is always exactly one tick, regardless of fall speed. On 128 tick that is 7.8125ms, the same size as CS2's intended `sv_bhop_time_window`. At a 16ms gap on 128 tick, `T = 2.048`, so the chance is `1 / 2.048 = 48.8%` at any fall speed.

## Jump cooldown
The chance that a press `g` ms after the previous one counts:

```
chance = clamp(g / t - 1, 0, 1)
```

The second press only counts if a whole input packet between the two presses has no press in it. That happens with probability `g / t - 1` for gaps between one and two ticks.

For example, on 128 tick a press 10ms after the previous one counts with probability 10 / 7.8125 − 1 = 0.28 (28%), and a press 12ms after counts with probability 12 / 7.8125 − 1 = 0.54 (54%). Anything under 7.8ms counts 0% of the time, and anything over 15.6ms counts 100% of the time.

# CS2 formulas

## Time spent in the last 2 units
The player reaches 2 units above the floor with speed `sqrt(v² - 2·g·2)`. Under constant gravity, the time from there to the floor is:

```
d = 64 · (v - sqrt(v² - 4g)) / g       (in ticks)
```

This is slightly longer than `2 / v` because the player is slower at the start of those 2 units. For example, `d = 0.506` tick (7.9ms) at 256 u/s and `d = 1.054` ticks at 128 u/s. Setting `d = 1` and solving for `v` gives 134.25 u/s, the landing speed at which the last 2 units take exactly one tick. The `2 / v` shortcut would give 128 u/s.

Plugging in 256 u/s: sqrt(256² − 3200) = sqrt(62336) = 249.67, so d = 64 × (256 − 249.67) / 800 = 0.506 tick, or 7.9ms. With the 2/v shortcut it would be 2/256 s = 0.5 tick (7.8ms).

## Window size for one landing
For a single landing with a given `a`, these are the lengths of the ranges of press times that give each outcome. They assume no other inputs near the landing and a landing above max speed.

**Case A: a tick ends inside the last 2 units (`a < d`).** The player is grounded while hovering at that tick end, and the landing time becomes that tick end.

| Outcome | Presses between | Length |
|---|---|---|
| Frame perfect | entering the last 2 units → tick end | `a` |
| Buffered perfect | (tick end − `w`) → entering the last 2 units | `max(0, w - a)` |
| Restored | tick end → tick end + `w` | `w` |

**Case B: the player touches the floor before the next tick end (`a >= d`).** The landing time is the moment of contact.

| Outcome | Presses between | Length |
|---|---|---|
| Frame perfect (in the last 2 units) | entering the last 2 units → contact | `d` |
| Frame perfect (after contact) | contact → min(tick end, contact + `w`) | `min(w, a - d)` |
| Buffered perfect | (contact − `w`) → entering the last 2 units, but not before the previous tick end | `max(0, min(w - d, 1 - a))` |
| Restored | tick end → contact + `w`, if the tick end comes first | `max(0, d + w - a)` |

Adding them up:

```
perfect(a) = max(a, w)                                          if a < d
           = d + min(w, a - d) + max(0, min(w - d, 1 - a))       if a >= d

total(a)   = max(a, w) + w                                      if a < d
           = d + w + max(0, min(w - d, 1 - a))                   if a >= d
```

`total` is perfect + restored, i.e. every jump that keeps the player's speed.

For example, at 256 u/s (d = 0.506 tick), a landing with a = 0.3 is Case A. Its perfect window is max(0.3, 0.25) = 0.3 tick (4.7ms, all frame perfect), its restored window is 0.25 tick, and its total is 0.55 tick (8.6ms). A landing with a = 0.1 is also Case A, and its perfect window is max(0.1, 0.25) = 0.25 tick, made of 0.1 tick of frame perfect presses and 0.15 tick of buffered presses. A landing with a = 0.8 is Case B. Its perfect window is 0.506 + min(0.25, 0.294) + 0 = 0.756 tick (11.8ms), its restored window is max(0, 0.506 + 0.25 − 0.8) = 0, and its total is also 0.756 tick.

## Average window size vs fall speed
These are the curves in `cs2_window_vs_fall_speed.png`: the averages of the formulas above over `a` from 0 to 1.

**Hard landings, `d < w` (faster than about 514 u/s):**
```
perfect  = d - d² + w - w²/2 + (w - d)²/2 + (w - d)(1 - w)
buffered = w·d - d²/2 + (w - d)²/2 + (w - d)(1 - w)
total    = 2w·d + (1 - d)(d + w) + (w - d)²/2 + (w - d)(1 - w)
```

**Normal landings, `w <= d <= 1 - w` (about 175 to 514 u/s):**
```
perfect  = d + w - d²/2 - w·d
buffered = w²/2
total    = d + w - (d² - w²)/2
```

**Soft landings, `1 - w < d <= 1` (about 134 to 175 u/s):**
```
perfect  = (1 + w²)/2            = 0.531 tick (8.3ms)
buffered = w²/2
total    = d + w - (d² - w²)/2
```

**Very soft landings, `d > 1` (slower than about 134 u/s):**
```
perfect  = (1 + w²)/2            = 0.531 tick (8.3ms)
buffered = w²/2                  = 0.031 tick (0.5ms)
total    = (1 + w²)/2 + w        = 0.781 tick (12.2ms)
```

Plugging in one fall speed from each range: at 1000 u/s (d = 0.128, hard), perfect = 0.429 tick (6.7ms), buffered = 0.123 tick (1.9ms), and total = 0.493 tick (7.7ms). At 256 u/s (d = 0.506, normal), perfect = 0.506 + 0.25 − 0.128 − 0.127 = 0.502 tick (7.8ms) and total = 0.756 − (0.256 − 0.0625)/2 = 0.659 tick (10.3ms). At 160 u/s (d = 0.827, soft), perfect = 0.531 tick (8.3ms) and total = 1.077 − (0.683 − 0.0625)/2 = 0.766 tick (12.0ms). At 100 u/s (d = 1.403, very soft), the values are the constants above, 0.531 tick perfect and 0.781 tick total.

The perfect window stops growing at `d = 1 - w` because from there on, the frame perfect range in both cases is always "entering the last 2 units → next tick end", which is `a` long whatever `d` is.

## Perfect bhop chance
These are the curves in `cs2_rate_vs_fall_speed.png` and `scroll_rate_comparison.png`.

With a steady scroll every `T` ticks and a random scroll phase, a range of press times that is `W` long contains a press with probability `min(1, W / T)`. This has to be applied to each landing (each `a`) before averaging, not to the average window:

```
chance = average over a of  min(1, window(a) / T)
```

**Perfect chance.** `perfect(a)` is never longer than 1 tick, and `T` is always longer than 1 tick (because of the cooldown), so the cap never applies:
```
perfect chance  = perfect / T
buffered chance = buffered / T
```

**Perfect + restored chance.** `total(a)` can be longer than `T` for soft landings, so the part above `T` has to be removed first:
```
excess = ½ · max(0, min(d, 1) + w - T)²  +  (1 - d) · max(0, d + w - T)     if w <= d < 1
       = ½ · max(0, 1 + w - T)²                                               if d >= 1
       = 0                                                                     if d < w

perfect + restored chance = (total - excess) / T
```

The first term is Case A with `a + w > T`. The second is Case B, where the window is `d + w` long for every `a` between `d` and 1. Without the excess term the chance is overstated for slow falls, by about 2.5 points at 128 u/s with a 16ms scroll.

For example, with a 16ms scroll (T = 1.024): at 256 u/s the perfect chance is 0.502 / 1.024 = 49.0%. Its total window never goes past d + w = 0.756 tick, which is shorter than T, so there is no excess and the perfect + restored chance is 0.659 / 1.024 = 64.4%. At 160 u/s, d + w − T = 0.053, so excess = ½ × 0.053² + (1 − 0.827) × 0.053 = 0.0014 + 0.0092 = 0.0105, and the perfect + restored chance is (0.766 − 0.0105) / 1.024 = 73.8%. At 128 u/s or slower (d ≥ 1), excess = ½ × (1.25 − 1.024)² = 0.0255, and the chance is (0.781 − 0.0255) / 1.024 = 73.8%, where leaving out the excess would give 76.3%.

**Against scroll speed** (`scroll_rate_comparison.png`), the same formulas are used with `v` fixed at 256 u/s (so `d = 0.506` tick) and `T` varying:
```
chance = 0                          if the gap is 15.625ms or less (cooldown)
chance = lottery, see below         between 15.625ms and 15.87ms
chance = formulas above             from 15.87ms
```

## Jump cooldown in CS2
A press is ignored if it comes 15.625ms or less after the previous press (ignored presses still reset the timer). Both press times are first rounded to 1/64 tick (0.244ms), so for a gap of `g` ms:

```
chance the press counts = clamp((g - 15.625) / 0.244, 0, 1)
```

A gap between 15.625ms and 15.87ms rounds to exactly one tick (ignored) or one tick plus 1/64 (counts), depending on where the presses fall on the 1/64 grid. The chance of rounding up is the fraction of the 0.244ms step the gap covers.

For example, a gap of 15.7ms counts with probability (15.7 − 15.625) / 0.244 = 0.31, a gap of 15.8ms with probability 0.72, and anything from 15.87ms up always counts.

For a steady scroll in that range, the gaps round to 64 or 65 units in a repeating pattern, and only the presses after a 65 count. At 15.7ms that is about 31% of presses, which left 37% of landings without any counted press after entering the last 2 units.

## Reference values

**If the landing time were correct, but early presses were still forgotten:** the landing would always be the moment of contact, and a press before contact would only count within the same tick. With `c` the position of the contact within its tick:
```
window(c) = min(w, c) + w                     (before contact + after contact)
average   = 2w - w²/2 = 0.469 tick (7.3ms)
```

**If both worked as intended:** `2w = 0.5` tick (7.8ms), the same as one 128 tick.

# How to derive the CS2 formulas yourself

Find, for one landing, every range of press times and which outcome each range gives, then average over where in the tick the landing happens.

**Step 1: pick the units and the random variable.** Measure time in ticks. The only thing that changes between two landings from the same height is where the tick grid falls relative to the fall. The cleanest way to describe that is `a`, the time from entering the last 2 units to the next tick end, which is uniform between 0 and 1.

As a running example, take a landing at 256 u/s with `a = 0.2`: the next tick ends 0.2 tick after the player enters the last 2 units.

**Step 2: get the fall timing from the physics.** Under constant gravity `g`, a player that reaches the floor at `v` reached 2 units above it with speed `sqrt(v² - 4g)`, so it took `d = (v - sqrt(v² - 4g)) / g` seconds to fall the last 2 units (multiply by 64 for ticks). With `B` the moment of entering the last 2 units, `C = B + d` is contact and `B + a` is the next tick end.

For the example, `d = 64 · (256 − sqrt(256² − 3200)) / 800 = 0.506` tick. Measuring from `B = 0`, contact is at `C = 0.506` and the next tick end is at `0.2`.

**Step 3: write down the rules from the code.** Every rule used here comes from reading the movement code:
1. The tick is split into subtick moves at every input and at every tick end. Each subtick move checks the jump button at its start, moves the player, then checks for ground at its end.
2. At the end of a subtick move, the player is put on the ground if there is floor within 2 units below them.
3. The landing time is the moment of contact if the player hit the floor during the move, and the end of the move if they were grounded while hovering.
4. A jump counts as a bhop if the press is within `w` of the landing time, i.e. in [landing − `w`, landing + `w`).
5. If the jump fires before any `WalkMove` after landing, it's a perfect bhop. If a `WalkMove` ran first, it's restored.
6. A press made while in the air is remembered, but thrown away at the start of the next subtick move unless it's within the window of the *current* last landing. While still in the air, that is the landing before the jump.

**Step 4: find the grounding moment.** The player is grounded at the end of the first subtick move that ends while they're within 2 units (rule 2) or that contains the contact (rule 3). With only tick ends and jump presses as boundaries, that is:
- the tick end, if it comes before contact (`a < d`), and no jump press came first
- the jump press itself, if the press is between entering the last 2 units and that tick end
- otherwise, the end of the subtick move that contains the contact

In the example, the tick end (`0.2`) comes before contact (`0.506`), so this is Case A. Without an earlier press, the player is grounded while hovering at `0.2`, and the landing time is `0.2`. A press between `0` and `0.2` would ground the player at the press instead.

**Step 5: go through every possible press time and classify it.** For a single press at time `p`:
- **`p` before the last 2 units:** the player is in the air, so the press is remembered (rule 6). It survives only if no other boundary comes before the grounding, i.e. the grounding happens at the end of the same subtick move. It then counts as buffered if `p` is at most `w` before the landing time. In Case A that gives the range from (tick end − `w`) to entering the last 2 units, `max(0, w - a)`. In Case B, contact is `d` after entering the last 2 units, so it only exists if `d < w`, and it can't start before the previous tick end: `max(0, min(w - d, 1 - a))`.
- **`p` inside the last 2 units and before the first tick end in there:** the press ends the subtick move, the player is grounded at `p`, the landing time is `p`, and the jump fires in the next subtick move before any `WalkMove`. That's frame perfect. This range is `a` long in Case A and `d` long in Case B.
- **`p` after contact, before the tick end (Case B):** the subtick move ending at `p` contains the contact, so the landing time is the contact and the press comes before `WalkMove`. It's frame perfect if `p - C < w`, so this range is `min(w, a - d)`.
- **`p` after a tick end that grounded the player:** `WalkMove` already ran in the subtick move from the tick end to `p`. If `p` is still within `w` of the landing time, it's restored.

In the example, going from the earliest press to the latest:
- `p = −0.03` (just before the last 2 units): the grounding at `0.2` ends the same subtick move, and `0.2 − 0.03 = 0.23 < 0.25`, so it is a buffered perfect bhop. Presses from `−0.05` to `0` work this way, a range of `w − a = 0.05` tick (0.8ms).
- `p = 0.1` (inside the last 2 units): the player is grounded at `0.1`, so it is frame perfect. Presses from `0` to `0.2` work this way, a range of `a = 0.2` tick (3.1ms).
- `p = 0.3` (after the tick end): `WalkMove` ran from `0.2` to `0.3`, and `0.3 − 0.2 < 0.25`, so it is restored. Presses from `0.2` to `0.45` work this way, a range of `w = 0.25` tick (3.9ms).
- `p = 0.6`: more than `w` after the landing time, so the jump only fires one tick after landing.

**Step 6: average over `a`.** Each length from step 5 is a simple function of `a`, so its average is an integral from 0 to 1 over `a`, split at the points where `max` and `min` switch (`a = w`, `a = d`, `a = d + w`, `a = 1 - (w - d)`). For example, in the normal range (`w <= d <= 1 - w`). In Case A, the frame perfect range `a` and the buffered range `max(0, w - a)` add up to `max(a, w)`. In Case B, the buffered term `max(0, min(w - d, 1 - a))` is zero because `d >= w`, so only the frame perfect ranges remain:

```
average perfect = ∫₀^d max(a, w) da  +  ∫_d^1 (d + min(w, a - d)) da
                = [w² + (d² - w²)/2]  +  [d(1 - d) + w²/2 + w(1 - d - w)]
                = d + w - d²/2 - w·d
```

For the example, this one landing has a perfect window of `0.2 + 0.05 = 0.25` tick and a total of `0.5` tick. Plugging `d = 0.506` into the average gives `0.506 + 0.25 − 0.128 − 0.127 = 0.502` tick perfect (7.8ms) over all landing positions at 256 u/s. The same integral done for the total window gives `d + w − (d² − w²)/2 = 0.756 − (0.256 − 0.0625)/2 = 0.659` tick (10.3ms) perfect + restored.

**Step 7: turn window lengths into chances.** A steady scroll with gap `T` and a random phase puts a press into a range of length `W` with probability `min(1, W / T)`. Apply this per landing, then average over `a`. If `W` never exceeds `T`, that's just `average W / T`. Otherwise, subtract the parts of the windows longer than `T` first (the `excess` term).

For the example with a 16ms scroll (`T = 1.024`), this landing gives a perfect bhop with probability `0.25 / 1.024 = 24.4%` and keeps speed with probability `0.5 / 1.024 = 48.8%`. Averaged over all landing positions at 256 u/s, the chances are `0.502 / 1.024 = 49.0%` perfect and `0.659 / 1.024 = 64.4%` perfect + restored.

**Step 8: check it.** A small simulation that places the presses and the tick grid explicitly and applies the rules from step 3 should give the same numbers as the formulas. The in-game measurements in the main document (about 130,000 landings) match within about 1.5 points. The remaining difference comes from the 1/64 tick rounding of press times, which the continuous formulas leave out: when one edge of a window sits exactly on the rounding grid and counts as inside, the window is effectively half a grid step (0.12ms) wider.

For the example, the in-game measurement at 256 u/s with a 16ms scroll was 49.2% perfect and 62.9% perfect + restored, against 49.0% and 64.4% from the formulas.
