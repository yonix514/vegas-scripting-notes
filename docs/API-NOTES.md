# VEGAS Pro 20 — verified scripting pitfalls

All five entries below were observed on **VEGAS Pro 20, build 411**, on Windows.
Each gives the failing code, what was observed, the working code, and how the
failure was detected. Nothing here is inferred from a type name.

---

## 1. Add first, configure after

### Failing code

```csharp
var k = new VideoMotionKeyframe(project, ev.Start);
k.Bounds = new VideoMotionBounds(tl, tr, br, bl);   // set BEFORE insertion
motion.Keyframes.Add(k);
```

**Observed:** on a run writing 288 animated events, only **72** actually ended
up with a usable keyframe — **216 lost**.

### Working code

```csharp
var k = new VideoMotionKeyframe(project, ev.Start);
motion.Keyframes.Add(k);                            // object exists in its container
k.Bounds = new VideoMotionBounds(tl, tr, br, bl);   // now configure it
```

**Observed:** **288 / 288**.

### Mechanism

Vegas computes a default value **at insertion time**, and that default
overwrites any value assigned before the insertion. The object must **exist in
its container** before it is configured.

Note the shape of this bug: it is **silent**. The C# is valid, no exception is
thrown, the property setter runs, and the value is discarded. You cannot find
it by reading the code.

### How it was found

By **re-reading the output**: counting how many events carried keyframes after
the run and comparing against how many were written. Not by inspecting code.

> **The rule worth keeping:** a script that writes must be paired with a script
> that re-reads. "The script ran" is a declaration, not evidence.

### Generalisation

Any API that offers `Add`/`Insert`/a factory is telling you the **container**
owns the initial state. Configure after insertion, not before.

---

## 2. `Timecode` units are not what the name suggests

```csharp
Timecode.Nanos                       // NOT nanoseconds
double ms = Timecode.Nanos / 10000.0; // ticks of 100 ns  ->  milliseconds

new Timecode(4000.0).ToMilliseconds()  // -> 4000
// the constructor takes MILLISECONDS, despite "Nanos" being the tick unit
```

**Verify by round-trip, never by name.** Construct with a known value, read back
the converted value, compare. This takes one line and settles the question
permanently:

```csharp
Console.WriteLine(new Timecode(4000.0).ToMilliseconds());   // expect 4000
```

---

## 3. `VideoMotionBounds` is in **source pixels**

`VideoMotionBounds` is expressed in **source pixels of the image as
pre-scaled to 1920 width**. The factor is:

```
f = W_src / 1920
```

These are **not** output pixels, and **not** normalised 0–1 coordinates.

### Frame-fill formula

For a source of `W × H` and an output ratio `R = W_out / H_out`:

```
if W/H > R:  window = (H × R)  × H        # source wider than the frame
if W/H < R:  window = W × (W / R)         # source taller than the frame
centre = (W/2, H/2)
```

**Clamp the centre**, or you get black bars:

```
centre ≥ half_window   AND   centre ≤ dimension − half_window
```

---

## 4. `VideoMotion.ScaleToFill` — a misleading name

`ScaleToFill` is **not** the UI's "match output format" option. It is a **local
boolean**: *"is this cropped region enlarged to fill the frame?"*

```
Bounds       = WHICH PART of the image is visible
ScaleToFill  = whether that part is ENLARGED to fill the frame
BOTH ARE REQUIRED.
```

**Observed symptom of getting it wrong:** with `Bounds` set correctly and
`ScaleToFill` left `false`, the cropped region keeps its own size and the
preview shows **black bars** — which reads as "my bounds are wrong" when the
bounds are fine.

### Settle a naming question with reflection, in one line

```csharp
var t = typeof(ScriptPortal.Vegas.VideoMotion);
var p = t.GetProperty("ScaleToFill",
    System.Reflection.BindingFlags.Public | System.Reflection.BindingFlags.Instance);
Console.WriteLine(p);   // Property Boolean ScaleToFill {get;set;}
```

A single reflection line beats an argument about vocabulary.

---

## 5. Other verified traps

```
TrackEvent.Split(t)
    t is an OFFSET from the event start, NOT an absolute timeline position.

Effect objects
    An Effect instance can belong to ONE event only.
    -> construct a new Effect per event, or you get an "unknown plug-in" error.

StringFormatFlags.NoWrap
    Crops the TOP of glyphs when the drawing box is shorter than the font.
    A capital "O" renders as a "U".

A .cs without its .dll does not run
    Vegas must compile it (csc.exe). Until then the script simply is not there.
```

---

## The procedure that finds these

```
1. LOCATE the actually-installed assembly (absolute path).
2. DUMP the signatures by reflection  -> keep the text file.
3. LIST runtime object names if the API exposes a plug-in tree.
4. WRITE a minimal probe on ONE object, then RE-READ the result.
5. COMPILE before running (csc.exe).
   A compile error costs nothing; an error inside Vegas costs a project.
6. SAVE THE PROJECT before any script that writes.
7. APPLY at scale.
8. RE-READ to verify: the writing script and the reading script are two scripts.
9. WRITE DOWN the traps you found.
```

Step 5 matters more than it looks. Compiling outside Vegas turns a class of
runtime surprises into compile errors, which are free.
