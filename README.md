# stone-scripts

Stonescript for [Stone Story RPG](https://stonestoryrpg.com/), loaded into the mobile game straight
from this repo with `import`.

## Layout

| File | |
|---|---|
| `main.txt` | The Mind Stone script itself. |
| `UI/MindstoneButton.txt`, `Cosmetics/WitchBroom.txt`, `Hats/WitchHat.txt` | Copies of the official library scripts `main.txt` imports, taken unchanged from `stonestoryrpg.com/stonescript/`. They have to live here because once the game is pointed at this repo, *every* `import` resolves against it. They do not update themselves; re-download them to pick up upstream changes. |

## The loader

Put this in the Mind Stone, and nothing else:

```
var urlSet = false
?!urlSet
  var s1="https://raw.githubusercontent.com/"
  var s2="shguan10/stone-scripts/"
  var s3="main/"
  sys.SetFileUrl(string.Format("{0}{1}{2}",
  ^s1,s2,s3))
  urlSet = true
//import main
```

- The URL is split over three variables because the editor is too narrow for it on one line.
- `urlSet` makes `sys.SetFileUrl` run once per run instead of every frame.
- `s3` can name any branch (`"some-branch/"`) to try out a script before merging it.
- The folder URL on its own answers 404. That is normal: GitHub raw serves files, not listings,
  and the game appends `<name>.txt` itself.

## The comment trick — every time the app starts

1. Start a run with the import **commented out** (`//import main`). This sets the URL. Return to
   Locations.
2. Remove the `//` and start another run. `main` loads.
3. Before closing the app, comment the import out again, so the next launch starts at step 1.

Why: the game resolves every `import` **before** any line of the script runs, and the URL resets
to the default on every launch. So with the import active, the first run looks in the default
location instead of here — and a failed import is remembered for the rest of the session. Every
later run then fails too with `Incomplete parsing main`, even once the URL is set, until the app
is closed. Run 1 has to set the URL with no import in the way.

On Android the default location is a folder inside the installed app
(`jar:file:///data/app/.../base.apk!/assets/Stonescript`), so a local `Stonescript` folder on the
phone's storage, as on desktop, is never read. Loading from a URL is the only way to add scripts.

## Caching

- A script, once loaded, is **cached for the rest of the app session**. Pushing a change does
  nothing until the app is closed and reopened (and the comment trick redone); then the new
  version loads under the same name.
- **`sys.cacheRemoteFiles = false` does not change this** — tested, the old version stayed.
  Leave it out.
- GitHub's raw CDN can itself serve the previous version for up to ~5 minutes after a push. If a
  change does not show after a restart, wait a few minutes and restart again.
- The editor's shortcut beside an `import` line always opens `stonestoryrpg.com/stonescript/`,
  whatever the URL is set to. Ignore it.

Tested on Android (OnePlus Nord N200, Android 12) with game version 4.33.0. The comment trick
matches what iOS players report.
