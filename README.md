# SampleNex

A deliberately tiny ZX Spectrum Next app whose only job is to **demo the
one-click build → push → run-on-hardware loop** from VS Code.

It prints one line:

```
Hello Dev Builders 42!
```

…where the number is bumped by **every single build**. That is the whole
trick: push it, read the number on the TV, build and push again, and the
number proves the bytes running on the Next are the ones you compiled
seconds ago — not a stale copy left on the SD card.

Built with [z88dk](https://z88dk.org) (`+zxn`, `.nex` output). Pushed with
[ZX-Next-Unite](https://github.com/jclauzel/ZX-Next-Unite)'s NextSync HTTP
bridge, driven by the `ZxNextRemote` PowerShell layer vendored in `tools/`.

---

## The demo, in one key

With everything set up (below), in VS Code press **`Ctrl+Shift+B`**:

1. `build.ps1` bumps `build_number.txt`, regenerates `src/build_number.h`,
   and compiles `SampleNex.nex`.
2. `tools/PS-Send-ToNext.ps1` waits for a Next on the bridge, uploads the
   `.nex`, and **verifies** it — comparing size and a 16-bit checksum read
   back *off the Next*, so an HTTP 200 alone is never mistaken for success.
3. `/forceexit` tells the Next to leave the Listener; the autoexec loop
   picks up the pushed file and runs it.
4. The Next prints `Hello Dev Builders <n>!`. Press **ENTER** and it drops
   back into the Listener, ready for the next push.

Edit, `Ctrl+Shift+B`, watch. That is the pitch.

## Layout

| Path | What |
|---|---|
| `src/main.c` | the app — a `printf` and a pause |
| `src/build_number.h` | **generated** each build (git-ignored) |
| `build_number.txt` | the counter, **tracked** so a clone continues the sequence |
| `build.ps1` | find z88dk → bump → generate → `zcc` → report |
| `.vscode/tasks.json` | Build / Send / Build+Send / autoexec / Clean |
| `Send-ToNext.cfg` | where the bridge is, what to push, where it lands |
| `tools/` | the vendored push layer (see *Refreshing the tools* below) |

## Setup, once

**1. z88dk** — install it. `build.ps1` looks for `%Z88DK_DIR%\bin\zcc.exe`,
then `C:\z88dk\bin\zcc.exe`. Any current nightly with the `+zxn` target.

```powershell
.\build.ps1          # should end with: BUILT: SampleNex.nex (…)
```

**2. ZX-Next-Unite** on the PC:
* Settings → **Enable NextSync HTTP bridge** (port 80).
* NextSync tab → start the **Remote Explorer** listen server.
  (Or launch Unite with `-start-remote-explorer-listener` to do both at
  startup.)

**3. `Send-ToNext.cfg`** — set `bridge_ip` to the PC running Unite (*not*
the Next). It ships as `127.0.0.1`, which is right when Unite runs on this
same machine. Set `token` only if Unite's *Require bearer token* is on.

**4. The Next** — put a NextSync **Listener** on it (ZX Next Remote, or a
`.sync5 -listen` dot) dialled at the PC, then run the VS Code task
**`Next: deploy autoexec loop`** once. That installs the receiving loop
into `c:/nextzxos/autoexec.bas`, checksum-verified, without pulling the SD
card. `Next: autoexec Off` / `On` park and re-arm it later.

The Next-side setup is documented in full in ZX-Next-Unite's
[`extra/README.md`](https://github.com/jclauzel/ZX-Next-Unite/blob/main/extra/README.md)
("Push-to-hardware from VS Code").

## Building by hand

```powershell
.\build.ps1              # bump + compile
.\build.ps1 -NoBump      # recompile, same number
.\build.ps1 -Sdcc        # zsdcc instead of sccz80
.\build.ps1 -Clean       # remove artifacts
```

Non-zero exit on failure, so the VS Code task fails loudly rather than
pushing a stale `.nex`.

## Exit codes (`PS-Send-ToNext.ps1`)

| Code | Meaning |
|---|---|
| 0 | pushed and **verified** on the Next |
| 1 | configuration problem |
| 2 | the bridge refused the bearer token |
| 3 | the operation failed (incl. a write refused by the remote machine's OS protection) |
| 4 | pushed, but **not verified** — the bytes on the Next differ |
| 5 | timed out waiting for a Next |

## Refreshing the tools

`tools/` is a **vendored copy** from ZX-Next-Unite's `extra/`, so this repo
is self-contained. Three things travel together — the script imports the
module relative to itself, and `-autoexec:Deploy` looks for `autoexec.bas`
beside it:

```powershell
$U = 'C:\path\to\ZX-Next-Unite\extra'
Copy-Item "$U\PS-Send-ToNext.ps1" tools\ -Force
Copy-Item "$U\ZxNextRemote"       tools\ -Recurse -Force
Copy-Item "$U\autoexec.bas"       tools\ -Force
```

The module's own reference is `Get-Help about_ZxNextRemote` (after
`Import-Module tools\ZxNextRemote\ZxNextRemote.psd1`), and the full guide
is ZX-Next-Unite's `PowerShell/PowerShellHelperClass.md`.

## Driving the Next from your own scripts

The same layer is a general client — useful when the demo grows:

```powershell
Import-Module .\tools\ZxNextRemote\ZxNextRemote.psd1

$bearer = ''                                   # '' = unprotected bridge
$remote = New-ZxNextRemote -IpAddress 127.0.0.1 -Port 80 -Token $bearer
$s      = $remote.ManageSession()              # the active Next

$s.Ls('/dev') | Format-Table
$s.Put('SampleNex.nex', '/dev/incoming.nex')
$s.Verify('SampleNex.nex', '/dev/incoming.nex')
$remote.Close()
```

## Licence

MIT - see [LICENSE](LICENSE).

`tools/` is vendored from
[ZX-Next-Unite](https://github.com/jclauzel/ZX-Next-Unite) (`extra/`),
MIT and the same author, so it travels under the same terms.
