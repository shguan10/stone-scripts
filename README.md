# stone-scripts

Stonescript for [Stone Story RPG](https://stonestoryrpg.com/), loaded into the mobile game straight
from this repo with `import`.

## Layout

| File | |
|---|---|
| `main.txt` | The Mind Stone script itself: runs the modules below in priority order. Set `debug` to print the target's id, state and time. |
| `lib/Gear.txt` | One frame's equip and ability requests. The first module to ask for a hand gets it; `Cast` keeps an ability's item held while it winds up. |
| `lib/Bosses.txt` | Mind Stone backflips out of each boss's avoidable attacks. Frame timings are community values; recalibrate here. |
| `lib/Harvest.txt` | Hatchet for trees, Shovel for the rest, and their abilities only while harvesting. |
| `lib/Travel.txt` | Out of combat: Quarterstaff dash (never at a boss), else Triskelion + Ouroboros (Star Stone near pickups). |
| `lib/Combat.txt` | Mask (bosses only), Skeleton Arm and Blade abilities whenever off cooldown; one the game will not fire goes back after 5 frames and is retried every 10 s. A wand and shield of the countering element; against a boss missing a debuff, the wand that applies it replaces the shield. A boss out of wand range is walked to only until it is first hit, so a backflip is not undone. Crossbows and melee are commented out in `Run`. |
| `lib/Melee.txt` | Boss fights: Bashing (else Dashing) Shield to close in, then each boss's best melee weapons (the list is at the top of the file). Two hands never hold the same item, so the second hand gets a different element. |
| `lib/Element.txt` | Which element beats the target, else the location. |
| `lib/Debuff.txt` | Against bosses: the weapon (sword, else wand) that applies whichever of weaken, burn and chill is missing. |
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

## Debugging a fight from a screen recording

With `debug = true` in `main.txt`, the top-left of the screen shows what the script sees: both
hands, the target's id/state/time, its debuffs, the last dodge and the last ability cast. Recording
a run and reading those lines frame by frame is the fastest way to find out why a fight went wrong.
Everything below runs on the computer the phone is plugged into (USB debugging on).

**Record.** The phone's own `screenrecord` is Permission denied on this ROM, so use
[scrcpy](https://github.com/Genymobile/scrcpy) (`sudo apt install scrcpy`; 1.25 tested):

```bash
timeout -s INT 420 scrcpy --no-display --record run.mkv --max-size 1200 --max-fps 30
```

Start it, then start the run. It stops by itself after 7 minutes; to stop early,
`pkill -INT -x scrcpy`. Matroska rather than MP4 so an interrupted file is still playable.

**Find the fight.** One frame every 5 seconds, timestamped, on one sheet:

```bash
ffmpeg -v error -y -i run.mkv -vf "fps=1/5,scale=480:-1,drawtext=text='%{pts\:hms}':x=5:y=h-20:fontsize=16:fontcolor=yellow:box=1:boxcolor=black,tile=4x12" -frames:v 1 overview.png
```

**Read the debug lines.** Where the overlay sits in the 1200×544 frame varies between
recordings (the text moved 130 px right between two runs of the same dungeon), so find the rows
first. This grabs the top of the frame at 2:05, doubled, with a red line every 10 px:

```bash
ffmpeg -v error -y -ss 125 -i run.mkv -vf "crop=800:110:0:0,scale=1600:-1,drawgrid=w=9999:h=20:color=red@0.6" -frames:v 1 rows.png
```

Then crop those rows out of a stretch of video. In the last recording the hands were
`crop=520:14:130:22` and `cast:` was `crop=300:14:130:100`; this shows both for every frame from
2:01 to 2:06, numbered, so you can count how many frames an item was held:

```bash
ffmpeg -v error -y -ss 121 -t 5 -i run.mkv -filter_complex "fps=30,split[a][b];[a]crop=520:14:130:22[t];[b]crop=300:14:130:100[c];[t][c]hstack,drawtext=text='%{n}':x=790:y=0:fontsize=12:fontcolor=yellow:box=1:boxcolor=black,tile=2x75" -frames:v 1 strip.png
```

For a slower overview of the same lines, use `fps=2` and a longer `-t`; your HP is on the bottom
row of the frame.

Timestamps and frame numbers in a strip count from `-ss`. For a hit itself, `fps=10` on the whole frame shows which
attack landed: the screen dims on the frame you take damage.
