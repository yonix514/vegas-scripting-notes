# Dumping the real API surface by reflection

The installed `ScriptPortal.Vegas.dll` is the only authority on what you can
call. Type names will not tell you member names, and they certainly will not
tell you **units** or **constructor argument order**.

## The dump format

Keep the output in plain text, one type per block, in a stable format:

```
### ScriptPortal.Vegas.VideoMotion  (base=Object) enum=False
   Property Boolean ScaleToFill  {get;set;}
   Property ScriptPortal.Vegas.VideoMotionKeyframes Keyframes  {get;}
### ScriptPortal.Vegas.VideoMotionKeyframe  (base=Object) enum=False
   Constructor Void .ctor(ScriptPortal.Vegas.Project, ScriptPortal.Vegas.Timecode)
   Constructor Void .ctor(ScriptPortal.Vegas.Timecode)
   Property ScriptPortal.Vegas.Timecode Position  {get;set;}
   Property ScriptPortal.Vegas.VideoMotionBounds Bounds  {get;set;}
   Property ScriptPortal.Vegas.VideoKeyframeType Type  {get;set;}
### ScriptPortal.Vegas.VideoMotionBounds  (base=Object) enum=False
   Constructor Void .ctor(VideoMotionVertex, VideoMotionVertex, VideoMotionVertex, VideoMotionVertex)
   Property VideoMotionVertex TopLeft  {get;set;}
   ... BottomRight, BottomLeft
```

## Three things that single dump teaches you

1. **`Keyframes` has a getter only.** You do not replace the collection, you
   **add to** it. That is already a hint toward the "add first, configure after"
   rule.
2. **`VideoMotionKeyframe` needs `(Project, Timecode)`** in one of its
   constructors. That is not derivable from the type name.
3. **`VideoMotionBounds` is built from four vertices** in the order
   **top-left, top-right, bottom-right, bottom-left**. The order is not
   discoverable from the signature, and getting it wrong mirrors or rotates
   your crop.

## Run the dump from inside Vegas

Reflection runs wherever the assembly is loaded. The reliable place is a
throwaway script launched from **Tools → Scripts**, so the assembly is already
loaded and its path is the real one:

```csharp
using System;
using System.IO;
using System.Reflection;
using System.Text;
using ScriptPortal.Vegas;

public class EntryPoint
{
    public void FromVegas(Vegas vegas)
    {
        var asm = typeof(Vegas).Assembly;
        var sb = new StringBuilder();
        sb.AppendLine("# assembly: " + asm.Location);
        sb.AppendLine("# " + asm.FullName);
        sb.AppendLine();

        foreach (var t in asm.GetExportedTypes())
        {
            sb.AppendLine("### " + t.FullName +
                          "  (base=" + (t.BaseType == null ? "-" : t.BaseType.Name) +
                          ") enum=" + t.IsEnum);
            foreach (var c in t.GetConstructors())
                sb.AppendLine("   Constructor " + c);
            foreach (var p in t.GetProperties())
                sb.AppendLine("   Property " + p);
            foreach (var m in t.GetMethods(
                         BindingFlags.Public | BindingFlags.Instance | BindingFlags.DeclaredOnly))
                sb.AppendLine("   Method " + m);
            sb.AppendLine();
        }

        var outPath = Path.Combine(Path.GetTempPath(), "vegas_api_dump.txt");
        File.WriteAllText(outPath, sb.ToString());
        System.Windows.Forms.MessageBox.Show(outPath);
    }
}
```

Compile line — see **[SCRIPT-FOLDERS.md](SCRIPT-FOLDERS.md)** for the full
reference list.

> **Keep the dump.** It is the reference you will come back to every time a
> property name lies about its unit. Regenerating it after a VEGAS upgrade is
> how you find behaviour that changed.
