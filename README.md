# Cutting a Godot web build from 119 MB to 37 MB

*Third edition: three projects, the export settings nobody chooses, a 7 MB file
nothing reads, and where the floor actually is.*

A teammate couldn't open our game on his phone. The loading bar stopped at 90%
and the screen went black. The same build booted fine on desktop, on all three
hosts. So it wasn't the environment and it wasn't boot code. It was weight.

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
same picture and is easier to measure, since it's the same encoded textures.

## 2. Lossless import inflates art that was already compressed

All 271 textures were importing at `compress/mode=0` (Lossless). That sounds
safe. What it actually does is throw away the compression the artist already
applied to the `.webp` and re-encode it, larger:

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
and on a web build shipped over mobile data, download size is the thing that
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

**Alpha was untouched**: a difference of exactly 0.00 across every file. So
there are no cutout or edge artifacts, which is the failure mode that actually
looks broken in a 2D game.

**The worst-looking case was invisible.** One texture showed banding in a dark
region, which looked alarming until I checked where those pixels were: 78.6% of
the differing pixels were fully transparent, pixels the game never draws. In
the visible pixels of that same region the mean difference was 4.78 out of 255,
under 2%.

### The caveat I could not rule out

If a shader samples the RGB of transparent pixels (a glow, an edge bleed, a
blur that reaches past the alpha edge), the garbage hiding in those transparent
pixels can surface. I didn't find one in this project, but a pixel diff can't
tell you this. It's the one thing that has to be checked on screen.

## 5. The result

```
.godot/imported   120 MB  ->  37 MB
```

Same art, same scenes, no assets deleted. The phone that couldn't load the game
loaded it.

## 6. The same method on two more projects

A year later, a different title in the same catalogue. Godot 4.7, Spine-based,
272 textures, and every one of them again at `compress/mode=0`. The default had
not changed and neither had the outcome.

```
index.pck        raw           brotli -q5
before      60,749,624       57,088,451
after       15,712,600       12,155,037
```

**74% less.** Switching the import mode alone took the pack from 60.7 MB to
16.3 MB, and `.godot/imported/` from 73 MB to 17 MB. Two projects, two years
apart, same default, same size of mistake.

Then a third, a smaller Spine-based game, and again every texture at
`compress/mode=0`:

```
index.pck        raw           brotli -q5
before      29,099,956       18,923,313
after       13,264,968        9,926,947
```

**48% less on the wire.** A smaller share this time because there was less
uncompressed art to begin with. The Spine atlases alone went from 26.1 MB of PNG
to 9.9 MB of lossless WebP, checked pixel by pixel against the originals, with no
scene touched.

That is the part worth taking away: this is not a story about one badly set up
project. Three projects, and the shipping default was there every time.

## 7. Your export ships things you never chose

The export preset in both projects was set to `all_resources`, which means
everything under the project directory goes into the pack whether a scene
references it or not. On the second project that was:

- `assets/_wip`, `assets/mockups`, `assets/thumbnails` — **3.4 MB** of design
  reference that belongs in the repository and not in the build
- three font families from a shared addon that no resource in this game
  referenced, because it uses its own copies
- `test_*.tscn` and `test_*.gd` — a test scene is not a game

All of it went into `exclude_filter`, none of it was deleted from the repo.

I have since watched someone else find the same shape independently. Ziwei Yu,
measuring [Island Evolution](https://islandevolution.com), found 71 stray test
screenshots, stored at 3x, swept into the pack by the same setting. It cost
10 MB: the pack went from 22 MB to 12 MB.

He then added the part that makes the fix hold: the deploy script fails if the
pack goes over 13 MB. An exclude filter only catches names it already knows; a
size gate catches the next thing nobody thought to name, at deploy instead of in
front of players.

**Treat it as a filter problem, not a cleanup problem.** Deleting the files
fixes today. An exclude rule fixes the next person who drops a screenshot into
the project folder.

## 8. A size audit finds dead things

Three textures made Godot's WebP re-encoder fail outright: `Failed decoding WebP
image`. I put them back to lossless, and they *still* produced no `.ctex`. They
had been broken before this change, silently, and nothing in any scene or script
referenced them. They were orphans that had been riding along in the repo.

On the third project the single largest file in the pack was
`extension_api.json`, **7.1 MB**. It is the dump of Godot's API that you compile a
GDExtension against. It arrived together with a plugin, nothing reads it at
runtime, and `all_resources` shipped it to every player anyway. If your project
uses GDExtension, search your pack for it before anything else: it may be the
biggest thing in there.

You tend to find a few of these whenever you sort a project by file size. It's a
good enough reason to do it once a year even when nothing is on fire.

## 9. The other levers, in order of payoff

Once textures were handled, the remaining budget was mostly fixed cost:

**Audio doesn't have to ship in the `.pck`.** Strip the audio directory at build
time and fetch it as a zip from a CDN into `user://` on first run. The game
starts before the sound is there and wires it up when it lands. This moves the
whole audio budget out of time-to-first-frame, which is what people actually
experience as "slow", and it's usually the second biggest block after textures.

**Spine atlases are usually PNG, and they do not have to be.** On the second
project the seven atlas pages were 8.46 MB of PNG. Converted to lossless WebP
they became 4.23 MB, exactly half, and the runtime loaded them without a change
to any scene. Lossless on purpose at the source: the only lossy step in the
chain should be Godot's import, so that you keep one knob instead of two.

**`icudt_godot.dat` is the price of internationalisation, and I measured it.**
Setting `locale/include_text_server_data=false` took the second project's pack
from 16,321,096 to 11,523,504 bytes. That is 4.8 MB, almost a third of what was
left after everything above.

I did not ship it. Without that dataset, Arabic bidirectional text and the line
breaking rules for Devanagari scripts degrade, and this is not something a
screenshot diff catches: the build gets smaller and one of your locales quietly
gets worse. Another title in the same catalogue runs without it in the same 12
languages, which proves it boots, not that its Arabic renders correctly.

The third project left it on for the same reason, until Arabic, Hindi and Nepali
are checked on screen.

**If you ship one locale, drop it and take the 4.8 MB.** If you ship Arabic,
Hindi or Nepali, this is the one saving on the list you should walk away from
until someone has looked at all three on a real screen. I wrote up what actually
breaks in [godot-i18n-that-holds-up](https://github.com/zednaked/godot-i18n-that-holds-up).

**Fonts add up fast when you support many scripts.** Four and a half megabytes,
in our case, across the fallbacks needed for non-Latin scripts. Subsetting is
the lever, and it is fiddly enough that I'd only reach for it after the two
above.

## 10. Where the floor is, and when to stop

Everything above is your pack. The engine ships alongside it and does not move
no matter what you do to your art. Measured on the second project:

```
                      on disk    brotli -q5
web.wasm               1.4 MB        0.5 MB
web.side.wasm         42.0 MB        8.1 MB
spine runtime          2.3 MB        0.3 MB
                                  ---------
engine floor                          8.9 MB
```

Two things follow from that table.

**The 42 MB is not what anyone downloads.** It is the figure people quote when
they panic about Godot on the web. Served with brotli, which any static host
does, it is 8.1 MB. If your host is not compressing the `.wasm` and the `.pck`,
that is one line of configuration and it is worth more than a week of asset
work.

**Below roughly 9 MB you are optimising the engine, not your game.** With the
pack at 12 MB brotli and the engine at 8.9, this build sits at about 21 MB and
further texture work has almost nothing left to give. Ziwei Yu's
Island Evolution, a completely unrelated Godot 4 game, measured 9.6 MB for its
engine and landed in the same place from the other direction.

Knowing where the floor is tells you when to stop, which is the part most size
advice never gets to. Past that point the lever is no longer bytes, it is what
the player looks at while those bytes arrive.

## The method, in five steps

1. **Unpack and sort by size.** Never optimise from intuition; 91% of our budget
   was in one line and none of the obvious suspects mattered.
2. **Check what your import settings are actually doing.** Lossless is not free,
   and it is not neutral. It re-encodes.
3. **Test the compression modes on a sample of your own art.** The right answer
   is art-dependent, and the popular answer was wrong for us by 54%.
4. **Quantify the loss where it's visible.** Diff against lossless, mask by
   alpha, and look at where the differing pixels actually are before rejecting a
   6x saving over a screenshot that looked bad.
5. **Check what your export ships that you never chose**, and fix it with a
   filter rather than a delete. Then find the engine floor and stop there.

---

Written from two production Godot 4.7 projects in a catalogue of commercial
titles shipped to the browser, where build size is a hard constraint rather than
a preference. The numbers are measured, not estimated, and every one of them
came out of a build that shipped.

### If your build has the same problem

I take this on as a fixed-scope audit: you send the exported build, I send back
the measurements, what is actually costing you, and the ordered list of what to
cut, largest saving first. No access to your source needed.

**USD 400 · 3 business days · no repo access**

**zednaked@gmail.com**

---

Other things I've made: [godot-canvas-shaders](https://github.com/zednaked/godot-canvas-shaders)
· [ZGT, a terminal inside the Godot editor](https://github.com/zednaked/zgt-bin)
· [mcpgodot](https://github.com/zednaked/mcpgodot)
