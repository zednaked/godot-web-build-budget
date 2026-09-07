# Cutting a Godot web build from 119 MB to 37 MB

A teammate couldn't open our game on his phone. The loading bar stopped at 90%
and the screen went black. The same build booted fine on desktop, on all three
hosts. So it wasn't the environment and it wasn't boot code — it was weight.

This is what I found, what I measured, and the one result that contradicted the
obvious guess.

## 1. Open the pack before you optimise anything

The temptation is to start deleting things. Don't. Unpack the `.pck` you're
actually serving and sort it by size. Ours, at 119.2 MB:

```
.ctex (textures)   108.59 MB   91.2%
icudt_godot.dat      4.58 MB    3.8%
fonts                4.54 MB    3.8%
everything else     ~1.40 MB    1.2%
```

One line was 91% of the budget. Every hour spent anywhere else would have been
wasted. If you don't want to unpack the `.pck`, `.godot/imported/` gives you the
same picture and is easier to measure — it's the same encoded textures.

## 2. Lossless import inflates art that was already compressed

All 271 textures were importing at `compress/mode=0` (Lossless). That sounds
safe. What it actually does is throw away the compression the artist already
applied to the `.webp` and re-encode it — larger:

```
side_panel.webp    source 0.61 MB  ->  5.02 MB packed   8.2x
cityscape.webp     source 1.03 MB  ->  5.43 MB          5.3x
background.webp    source 2.17 MB  ->  9.39 MB          4.3x
```

67.9 MB of source art was becoming 108.6 MB in the pack. The build was bigger
than the raw assets on disk.

This is easy to miss because nothing is *wrong*. Lossless is the default-looking
choice, it's the one that sounds careful, and the number it produces never
appears anywhere in the editor.

## 3. The obvious guess was wrong: VRAM compression made it bigger

Everyone reaches for VRAM Compressed for web. I tested both modes on five
textures before committing to either:

```
                 lossless (0)   lossy (1)   VRAM compressed (2)
sum of 5 files      28.19 MB      7.42 MB          43.33 MB
```

**VRAM compression made it 54% larger. Lossy cut 74%.**

The reason is that VRAM compression is fixed-rate: it spends the same bits per
pixel whether the texture is a photographic background or a flat two-colour UI
panel. For 2D games with large areas of simple art, that is the worst possible
trade. VRAM compression buys you GPU memory and decode speed, not download size
— and on a web build shipped over mobile data, download size is the thing that
was breaking.

Test it on your own art before you assume. Five files and ten minutes is enough
to see which way it goes.

## 4. Measure the visual loss instead of squinting at it

"Lossy" triggers an instinct to reject it. Quantify it instead. I diffed each
lossy texture against its lossless version pixel by pixel, counting **only where
the art is actually visible** (alpha > 32):

```
texture         mean diff (visible)   % of px > 8
background             6.40              27.7%
cityscape              3.77               8.5%
side_panel             4.08               7.6%
halftone               1.00               0.0%
popup                  7.08              30.6%
```

Two things mattered more than the averages:

**Alpha was untouched** — a difference of exactly 0.00 across every file. So
there are no cutout or edge artifacts, which is the failure mode that actually
looks broken in a 2D game.

**The worst-looking case was invisible.** One texture showed banding in a dark
region, which looked alarming until I checked where those pixels were: 78.6% of
the differing pixels were fully transparent — pixels the game never draws. In
the visible pixels of that same region the mean difference was 4.78 out of 255,
under 2%.

### The caveat I could not rule out

If a shader samples the RGB of transparent pixels — a glow, an edge bleed, a
blur that reaches past the alpha edge — the garbage hiding in those transparent
pixels can surface. I didn't find one in this project, but a pixel diff can't
tell you this. It's the one thing that has to be checked on screen.

## 5. The result

```
.godot/imported   120 MB  ->  37 MB
```

Same art, same scenes, no assets deleted. The phone that couldn't load the game
loaded it.

## 6. A size audit finds dead things

Three textures made Godot's WebP re-encoder fail outright: `Failed decoding WebP
image`. I put them back to lossless — and they *still* produced no `.ctex`. They
had been broken before this change, silently, and nothing in any scene or script
referenced them. They were orphans that had been riding along in the repo.

You tend to find a few of these whenever you sort a project by file size. It's a
good enough reason to do it once a year even when nothing is on fire.

## 7. The other levers, in order of payoff

Once textures were handled, the remaining budget was mostly fixed cost:

**Audio doesn't have to ship in the `.pck`.** Strip the audio directory at build
time and fetch it as a zip from a CDN into `user://` on first run. The game
starts before the sound is there and wires it up when it lands. This moves the
whole audio budget out of time-to-first-frame, which is what people actually
experience as "slow", and it's usually the second biggest block after textures.

**`icudt_godot.dat` (~4.5 MB) is the price of internationalisation.** It ships
when you use Godot's i18n machinery. Worth knowing it's there and that it is not
a leak — but if you ship one locale, check whether you need it at all.

**Fonts add up fast when you support many scripts.** Four and a half megabytes,
in our case, across the fallbacks needed for non-Latin scripts. Subsetting is
the lever, and it is fiddly enough that I'd only reach for it after the two
above.

## The method, in four steps

1. **Unpack and sort by size.** Never optimise from intuition; 91% of our budget
   was in one line and none of the obvious suspects mattered.
2. **Check what your import settings are actually doing.** Lossless is not free,
   and it is not neutral — it re-encodes.
3. **Test the compression modes on a sample of your own art.** The right answer
   is art-dependent, and the popular answer was wrong for us by 54%.
4. **Quantify the loss where it's visible.** Diff against lossless, mask by
   alpha, and look at where the differing pixels actually are before rejecting a
   6x saving over a screenshot that looked bad.

---

Written from a production Godot 4.7 project — a catalogue of commercial titles
shipped to the browser, where build size is a hard constraint rather than a
preference.

Other things I've made: [godot-canvas-shaders](https://github.com/zednaked/godot-canvas-shaders)
· [ZGT, a terminal inside the Godot editor](https://github.com/zednaked/zgt-bin)
· [mcpgodot](https://github.com/zednaked/mcpgodot)
