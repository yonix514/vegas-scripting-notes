# vegas-scripting-notes

**The VEGAS Pro scripting traps that are not in the documentation.**
Five verified pitfalls, the failing code, the measured failure, and the fix —
plus how to dump the real API surface instead of guessing.

---

## Why this exists

The `ScriptPortal.Vegas` API is a .NET assembly that ships without usable
reference documentation. The object model is discoverable, but the *behaviour*
is not. So you end up with failures that look like your code is wrong when in
fact the API accepted your code and quietly ignored it.

Three of the five pitfalls below cost real production time. One of them
silently produced **wrong output on 75 % of the frames** while every line of
code looked correct.

This is written for people who have to write a VEGAS script and do not want to
rediscover all of it.

---

## The pitfalls, in one table

| # | Pitfall | Observed failure | Fix |
| --- | --- | --- | --- |
| 1 | Setting `VideoMotionKeyframe.Bounds` **before** `Keyframes.Add()` | **72 / 288** frames animated | Add first, set after |
| 2 | `Timecode` units | value off by 10 000× | `new Timecode(ms)` takes **milliseconds**; `Nanos / 10000.0` gives ms |
| 3 | `VideoMotionBounds` coordinate space | crop window lands in the wrong place | coordinates are in **source pixels**, normalised to 1920 width |
| 4 | `VideoMotion.ScaleToFill` | black bars in the preview | it is **not** "match output format" — it is a local boolean, and you need it **plus** `Bounds` |
| 5 | `TrackEvent.Split(t)` | split at the wrong place | `t` is an **offset from the event start**, not an absolute position |

Details, with the failing and working code: **[docs/API-NOTES.md](docs/API-NOTES.md)**.

---

## Pitfall 1, in full

This is the one that produced wrong output while looking correct.

```csharp
// WRONG — 72 animated images out of 288
var k = new VideoMotionKeyframe(project, ev.Start);
k.Bounds = new VideoMotionBounds(tl, tr, br, bl);   // set BEFORE insertion
motion.Keyframes.Add(k);                            // Vegas overwrites it
```

```csharp
// RIGHT — 288 out of 288
var k = new VideoMotionKeyframe(project, ev.Start);
motion.Keyframes.Add(k);                            // the object EXISTS in its container
k.Bounds = new VideoMotionBounds(tl, tr, br, bl);   // now set it
```

**Why.** Vegas computes a default value **at insertion time** and overwrites
whatever you assigned before. The object must **exist in its container** before
you configure it.

**The general rule, which is worth more than the specific fix:** any API that
gives you a factory or an `Add` is telling you the container owns the initial
state. Configure after, not before. This is the same failure family as any
"silent default" you meet elsewhere — the code is accepted, the value is
discarded, and nothing raises an error.

**How it was caught.** Not by reading the code — it was correct C#. By
**re-reading the output**: counting how many events actually carried keyframes
at the end, and comparing against how many were written. 72 against 288.

That check is the real lesson: **a script that writes must be paired with a
script that re-reads.** "The script ran" is a declaration, not evidence.

---

## Pitfall 4 — the one with the misleading name

`ScaleToFill` sounds like the "match output format" option in the Vegas UI. It
is not.

```
Bounds       = WHICH PART of the image you see
ScaleToFill  = whether that part is ENLARGED to fill the frame
```

**Both are needed.** With `Bounds` set and `ScaleToFill` left `false`, the
cropped region keeps its own size and you get **black bars** — which reads as
"my bounds are wrong" when the bounds are fine.

If a name is ambiguous, **settle it with reflection in one line** rather than
arguing about vocabulary:

```csharp
var p = typeof(ScriptPortal.Vegas.VideoMotion).GetProperty("ScaleToFill",
    System.Reflection.BindingFlags.Public | System.Reflection.BindingFlags.Instance);
// -> Property Boolean ScaleToFill {get;set;}   : a local boolean, not a mode
```

---

## Stop guessing: dump the real API

You cannot guess member names or units from a type name. Dump the assembly that
is actually installed and read the signatures.

→ **[docs/REFLECTION.md](docs/REFLECTION.md)**

Two findings from a single dump, both impossible to guess:

- `Keyframes` has a **getter only** — you do not replace the collection, you
  **add to** it. That alone hints at the pitfall-1 rule.
- `VideoMotionBounds` is constructed from **four vertices** in the order
  **top-left, top-right, bottom-right, bottom-left**. The order is not
  discoverable from the constructor signature.

---

## Where the scripts go, and how they compile

→ **[docs/SCRIPT-FOLDERS.md](docs/SCRIPT-FOLDERS.md)**

Short version: VEGAS reads more than one `Script Menu` folder, including one
under `%LOCALAPPDATA%` that is easy to miss; installing under
`C:\Program Files\VEGAS\...` needs administrator rights; and the script will not
run until `csc.exe` has compiled it, which needs
`ScriptPortal.Vegas.dll` plus `System.Windows.Forms.dll`, `System.Drawing.dll`,
`System.Core.dll` and `Microsoft.CSharp.dll`.

---

## What I measured

| Claim | How it was measured | Result |
| --- | --- | --- |
| Set-before-add is discarded | counted events carrying keyframes after the run | **72 / 288** wrong order · **288 / 288** correct order |
| A `.cs` without its `.dll` does nothing | ran the script before compiling | no entry in the Scripts menu, no error |
| `new Timecode(4000.0)` is milliseconds | round-trip: construct, then `ToMilliseconds()` | **4000** returned, not 0.0004 |
| `ScaleToFill` is a local boolean | reflection over the installed assembly | `Property Boolean ScaleToFill {get;set;}` |

### One correction I have to make

An earlier version of my notes claimed that `ffmpeg`'s `acrossfade` filter
**"shortens every junction by 30 ms because of its default `overlap=1`"**, and
that this had cost 28.29 s over 943 junctions.

**I re-measured it, and that explanation is wrong.**

`acrossfade=d=D` shortens the total by **exactly `D` per junction** compared
with plain concatenation. That is the documented meaning of the crossfade
duration, not a hidden defect, and it is **identical for `overlap=0` and the
default `overlap=1`**. Measured on ffmpeg 7.1.5: two 2.000 s sources with
`acrossfade=d=0.5` give 3.500 s; `overlap=0` and `overlap=1` produce the same
duration. A 400-junction chain reproduced the expected arithmetic with **zero**
additional drift.

The **practical advice survives**: when you assemble audio from many segments,
verify the **output** duration against the target, and prefer plain
concatenation (or exact sample slicing) when you want no duration change at all.
The **mechanism** I published was wrong, so I am correcting it here rather than
quietly deleting it. Full measurements:
[ffmpeg-concat-audit](https://github.com/youssef-montage) *(repo to be published)*.

---

## Scope and honesty

- Everything here was observed on **VEGAS Pro 20, build 411**, on Windows.
- The API is stable across recent versions, so most of it should hold for
  VEGAS 18–22 — but "should" is not "verified", and I have not tested those.
- **No binary is shipped.** The `.dll` files are compiled per machine; shipping
  someone else's compiled assembly is both fragile and a good way to spread
  something you did not inspect.
- If a pitfall here stops being true in a newer build, an issue with the build
  number and the repro is genuinely useful.

## License

[MIT](LICENSE).

## Related

- [`vegas-script-toolkit`](https://github.com/youssef-montage) — 15 working scripts for the
  "images over voice-over" workflow, which is where these pitfalls were found.
