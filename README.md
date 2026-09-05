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
bridge, driven by its `ZxNextRemote` PowerShell layer.

> ### ⚠️ `tools/` is a vendored snapshot
>
> The PowerShell layer in [`tools/`](tools/) is a **copy**, taken from
> ZX-Next-Unite so this repo works standalone. It does **not** update
> itself, and it will fall behind.
>
> **ZX-Next-Unite is the upstream and the source of truth** — refresh from
> it whenever you hit a bug or want a new feature:
>
> * [`extra/`](https://github.com/jclauzel/ZX-Next-Unite/tree/main/extra) — the
>   `ZxNextRemote` module, `PS-Send-ToNext.ps1` and `autoexec.bas`
> * [`PowerShell/`](https://github.com/jclauzel/ZX-Next-Unite/tree/main/PowerShell) — the
>   guide, and the VS Code task sample these tasks came from
>
> Copy commands are in [*Refreshing the tools*](#refreshing-the-tools).

---

## The demo, in one key

With everything set up (below), in VS Code press **`Ctrl+Shift+B`**:

1. `build.ps1` bumps `build_number.txt`, regenerates `src/build_number.h`,
   and compiles `SampleNex.nex`.
2. `tools/PS-Send-ToNext.ps1` waits for a Next on the bridge, uploads the
   `.nex`, and **verifies** it — asking the Next for the **CRC-32 of the
   file it holds** (computed *on the Next*; 8 hex digits come back, not the
   file) and comparing it with the CRC-32 of the bytes pushed, so an HTTP
   200 alone is never mistaken for success. A Listener older than ZX Next
   Remote 1.0.8 / `.sync5` 5.9.2 does not know the crc op; the script then
   says so and falls back to the size + 16-bit checksum read-back.
3. `/forceexit` tells the Next to leave the Listener; the autoexec loop
   picks up the pushed file and runs it.
4. The Next prints `Hello Dev Builders <n>!`. Press **ENTER**: the app
   **soft-resets** the machine, which comes back up into the autoexec
   loop and re-arms the Listener, ready for the next push.

Edit, `Ctrl+Shift+B`, watch. That is the pitch.

## Layout

| Path | What |
|---|---|
| `src/main.c` | the app — a `printf`, a pause, then a soft reset |
| `src/build_number.h` | **generated** each build (git-ignored) |
| `build_number.txt` | the counter, **tracked** so a clone continues the sequence |
| `build.ps1` | find z88dk → bump → generate → `zcc` → report |
| `.vscode/tasks.json` | Build / Send / Build+Send / autoexec / Clean |
| `Send-ToNext.cfg` | where the bridge is, what to push, where it lands |
| `tools/` | **vendored snapshot** of the push layer — update it from upstream, see *[Refreshing the tools](#refreshing-the-tools)* |

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
into `c:/nextzxos/autoexec.bas`, verified the same way, without pulling the
SD card. The loop's transfer folder defaults to **`/home`** — the
`remote_path` in `Send-ToNext.cfg` matches it — and the loop is
configurable on the Next itself (hold **B** while it boots; the settings
live in `c:/nextzxos/autoexec.cfg`). The two halves must name the same
folder: a PC pushing into one while the Next watches another fails
silently. `Next: autoexec Off` / `On` park and re-arm the loop later.

The Next-side setup is documented in full in ZX-Next-Unite's
[`extra/README.md`](https://github.com/jclauzel/ZX-Next-Unite/blob/main/extra/README.md)
("Push-to-hardware from VS Code").

### Why it ends with a soft reset

Returning from `main` on this target means z88dk's "return to basic"
exit (`crt_on_exit = 0x30002`). On a machine sitting in the autoexec
loop that is the wrong ending: BASIC resumes the loop, the loop finds
the pushed file and `.nexload`s it **again**. On the first hardware test
that ran the app three times over, output piling down the screen, until
it fell over into garbage.

The loop is built around ZX Next Remote's habit of soft-resetting when
it exits — the reset *is* the "go to the top of the loop". So this app
ends the same way, via the Next's reset register
(`ZXN_WRITE_REG(REG_RESET, RR_SOFT_RESET)`), and the cycle closes
properly. It also means no `CLS` is needed on entry: the reset clears
the screen. (The old `printf("%c", 12)` was what printed a stray `?` at
the top of every line — this console driver has no form-feed and just
echoes the unknown character.)

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

**`tools/` is a vendored snapshot, and keeping it current is recommended.**
It is a copy taken from ZX-Next-Unite at the time this repo was written —
nothing here updates it, so bug fixes and new features land upstream first
and reach you only when you refresh. Upstream is:

| Upstream | Holds |
|---|---|
| [`ZX-Next-Unite/extra/`](https://github.com/jclauzel/ZX-Next-Unite/tree/main/extra) | the `ZxNextRemote/` module, `PS-Send-ToNext.ps1`, `autoexec.bas` — **the three files vendored here** |
| [`ZX-Next-Unite/PowerShell/`](https://github.com/jclauzel/ZX-Next-Unite/tree/main/PowerShell) | `PowerShellHelperClass.md` (the full guide) and `vscode-sample/tasks.json`, which this repo's `.vscode/tasks.json` is adapted from |

### How to refresh

Clone or pull ZX-Next-Unite, then copy the **three** pieces — they travel
together, because the script imports the module relative to itself and
`-autoexec:Deploy` looks for `autoexec.bas` beside it:

```powershell
# once: git clone https://github.com/jclauzel/ZX-Next-Unite.git
# then: git -C C:\path\to\ZX-Next-Unite pull

$U = 'C:\path\to\ZX-Next-Unite\extra'
Copy-Item "$U\PS-Send-ToNext.ps1" tools\ -Force
Copy-Item "$U\ZxNextRemote"       tools\ -Recurse -Force
Copy-Item "$U\autoexec.bas"       tools\ -Force

git status tools\        # anything listed = you just picked up changes
```

A refresh that reports nothing means the snapshot was already current.
After one that *does* change something, re-run a push to be sure your
setup still behaves — `.\build.ps1` then the **Send to Next** task.

**Refreshing the loop already on the card.** `Next: deploy autoexec loop`
never overwrites an existing `autoexec.bas`, so a card set up from an
earlier snapshot keeps running the loop it has. This repo's first
snapshot (before 28 August 2026) shipped a loop hard-wired to `/dev`; the
current one defaults to `/home` and is configurable on the Next. To move
such a card over, run **`Next: autoexec Off`** (parks the old loop) and
then **`Next: deploy autoexec loop`** (sends the new one and removes the
parked copy) — or keep the old loop and set
`remote_path = /dev/incoming.nex` in `Send-ToNext.cfg` again.

### Reference

* `Get-Help about_ZxNextRemote` — the class and method reference, after
  `Import-Module tools\ZxNextRemote\ZxNextRemote.psd1`.
* [`PowerShellHelperClass.md`](https://github.com/jclauzel/ZX-Next-Unite/blob/main/PowerShell/PowerShellHelperClass.md)
  — the full guide: install/uninstall, the error reference, the
  bearer-token pattern and the VS Code integration.

## Driving the Next from your own scripts

The same layer is a general client — useful when the demo grows:

```powershell
Import-Module .\tools\ZxNextRemote\ZxNextRemote.psd1

$bearer = ''                                   # '' = unprotected bridge
$remote = New-ZxNextRemote -IpAddress 127.0.0.1 -Port 80 -Token $bearer
$s      = $remote.ManageSession()              # the active Next

$s.Ls('/home') | Format-Table
$s.Put('SampleNex.nex', '/home/incoming.nex')
$s.Verify('SampleNex.nex', '/home/incoming.nex')   # CRC-32 on the Next; /sum fallback
$s.Crc('/home/incoming.nex').Crc32                 # the Next's digest, 8 hex digits
$remote.Close()
```

## Licence

MIT - see [LICENSE](LICENSE).

`tools/` is vendored from [ZX-Next-Unite](https://github.com/jclauzel/ZX-Next-Unite/tree/main/extra)
(`extra/`), MIT and the same author, so it travels under the same terms.
