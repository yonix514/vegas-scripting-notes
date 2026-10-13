# Where VEGAS looks for scripts, and how they compile

## The folders

VEGAS Pro 20 reads **several** `Script Menu` folders. The two that matter:

| Scope | Folder |
| --- | --- |
| All users on the machine | `C:\Program Files\VEGAS\VEGAS Pro 20.0\Script Menu\` |
| Your account only | `%APPDATA%\VEGAS\VEGAS Pro\20.0\Script Menu\` |
| Variant seen on some installs | `%LOCALAPPDATA%\VEGAS\VEGAS Pro\20.0\Script Menu\` |

Writing into `C:\Program Files\...` requires **administrator rights**. If you do
not have them, use the per-account folder.

**The `%LOCALAPPDATA%` one is easy to miss, and it is the folder that bites you
when a script you edited appears not to have changed** — you may be editing the
file in one folder while VEGAS is loading a compiled copy from another.

### Find what actually exists, before creating anything

```bat
dir "%APPDATA%\VEGAS" /s /b | findstr /i "Script Menu"
dir "%LOCALAPPDATA%\VEGAS" /s /b | findstr /i "Script Menu"
dir "C:\Program Files\VEGAS" /s /b | findstr /i "Script Menu"
```

If the folder does not exist, **create it**. VEGAS does not always create it for
you, and it **ignores a missing path silently** — which looks exactly like a
broken script.

For a detailed install walkthrough, see the companion repository
[`vegas-script-toolkit`](https://github.com/youssef-montage) →
`docs/INSTALLATION.md`.

## Compilation

VEGAS compiles a `.cs` on **first launch** from the Scripts menu. To see errors
properly, compile by hand:

```bat
"%WINDIR%\Microsoft.NET\Framework64\v4.0.30319\csc.exe" /nologo /target:library ^
  /out:"%TEMP%\MonScript.dll" ^
  /reference:"C:\Program Files\VEGAS\VEGAS Pro 20.0\ScriptPortal.Vegas.dll" ^
  /reference:System.dll ^
  /reference:System.Windows.Forms.dll ^
  /reference:System.Drawing.dll ^
  /reference:System.Core.dll ^
  /reference:Microsoft.CSharp.dll ^
  "MonScript.cs"
```

### Required references

| Reference | Needed for |
| --- | --- |
| `ScriptPortal.Vegas.dll` | the VEGAS API itself (`Vegas`, `TrackEvent`, `Timecode`, …) |
| `System.Windows.Forms.dll` | dialogs (`MessageBox`, `OpenFileDialog`) |
| `System.Drawing.dll` | geometry — `VideoMotionBounds`, vertices, colours |
| `System.Core.dll` | LINQ |
| `Microsoft.CSharp.dll` | needed by some `dynamic` expressions |

Adjust the `ScriptPortal.Vegas.dll` path to your version — replace `20.0` with
your own.

### The entry point

VEGAS looks for a class named `EntryPoint` with a public method:

```csharp
public class EntryPoint
{
    public void FromVegas(Vegas vegas) { /* ... */ }
}
```

Nothing else is called. If the method signature is wrong or the class is not
named `EntryPoint`, the script loads and does nothing — silently.

## Caching

After editing a script, if VEGAS seems to ignore the change, clear the compiled
cache:

```bat
rmdir /s /q "%LOCALAPPDATA%\VEGAS\VEGAS Pro\20.0\Script Menu"
```

Then restart VEGAS. **Save your project first.**
