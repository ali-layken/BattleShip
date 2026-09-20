# GameCube Adapter (WUP-028) — macOS Testing Handoff (2026-09-20)

Native GameCube adapter work: `JRickey/libultraship#8` + `JRickey/BattleShip#275`.
Linux and Windows are tested. **macOS is the only untested platform, and it is
the one where the design's known limitation can actually bite.**

There is no driver step on macOS — no Zadig, no WinUSB, nothing to install.
Plug the adapter in and it works as a normal HID device. That is exactly what
makes macOS interesting: the adapter is usable by *both* backends there, so
whichever one wins matters.

## TL;DR

- **Linux:** works (author, real WUP-028, system libusb 1.0.30).
- **Windows:** works with the WinUSB driver bound, including hotplug. Without
  it, the OS won't start the device at all, so nothing can use it.
- **macOS:** unrun. The open question is whether libusb can claim interface 0
  while Apple's HID driver owns the device.
- **The one test that matters most is Test 2.** If libusb *cannot* claim the
  adapter on macOS, this PR makes the adapter unusable on a Mac where it
  previously worked fine as a plain SDL gamepad — because the SDL ignore is
  registered unconditionally at startup. That would be a macOS regression and a
  merge blocker.

## Getting the right branches

**This will bite you first.** `.gitmodules` points `libultraship` at
`JRickey/libultraship`, but the pinned submodule SHA only exists on
**`ali-layken/libultraship`**, branch `feature/gc-adapter`, because PR #8 isn't
merged. A plain `git submodule update --init` fails to find the commit.

```bash
# superproject - ali-layken/BattleShip, branch feature/gc-adapter
git fetch origin
git checkout feature/gc-adapter          # -> 9051a41
git submodule update --init --recursive  # libultraship WILL fail here - expected

# point the submodule at the fork that actually has the commit
git -C libultraship remote set-url origin https://github.com/ali-layken/libultraship.git
git -C libultraship fetch origin
git -C libultraship checkout 1c6e5f1c    # tip of feature/gc-adapter
```

Verify before building:

```bash
git rev-parse --short HEAD                   # -> 9051a41
git -C libultraship rev-parse --short HEAD   # -> 1c6e5f1c
git submodule status libultraship            # no leading +/- once correct
```

Each branch is three commits: the feature commit, a follow-up, and a revert of
most of that follow-up. The **net** change against the feature commit is 21
lines in the no-libusb stub — nothing that compiles on macOS differs from what
was tested on Linux.

Do **not** commit a `.gitmodules` URL change. On the Windows box that edit was
made locally and deliberately kept out of every commit; the PR must keep
pointing at `JRickey/libultraship`.

## Building

```bash
cmake -S . -B build-cmake -GNinja
cmake --build build-cmake --target ssb64 -j 4    # cap at 4 on M1 16 GB
```

macOS never uses a system libusb — `src/CMakeLists.txt` only consults
pkg-config on Linux/BSD, so macOS always builds **libusb-cmake v1.0.30-0
statically**. Confirm at configure time:

```
-- GameCube adapter: libusb built from source (libusb-cmake)
```

If you instead see `libusb unavailable, native support disabled`, the driver
compiled to its stub and none of the tests below mean anything. Fix that first.

The binary needs `BattleShip.o2r` and `f3d.o2r` in its working directory:

```bash
cmake --build build-cmake --target ExtractAssets   # first time only
cd build-cmake && ./ssb64
```

Logs land in the working directory: `logs/BattleShip.log` (LUS/spdlog,
cumulative) and `ssb64.log` (port trace, overwritten each run). Every test
below is read out of the first one:

```bash
grep -iE 'gcadapter|0337|another input backend' logs/BattleShip.log
```

---

## Test 1 — does libusb claim the adapter at all?

Plug the adapter in **before** launching, then read the log.

On Linux the code leans on `libusb_set_auto_detach_kernel_driver()` to kick
`usbhid` off the interface. That call is Linux-only; on macOS it returns
`LIBUSB_ERROR_NOT_SUPPORTED` and the code ignores the result, so the claim at
`GCAdapter.cpp:138` has to succeed against whatever Apple's IOKit HID stack has
already matched.

**Claimed (good):**

```
[gcadapter] started (hotplug=true)
[gcadapter] adapter opened (in=0x81 out=0x02)
ConnectedPhysicalDeviceManager: globally ignoring SDL gamepads 057e:0337 (claimed by another input backend)
[gcadapter] port 0 controller connected (type=1, origin=128,127 c=128,127)
```

Expect `hotplug=true` on macOS — libusb supports hotplug on darwin. Windows
reports `false` and falls back to a one-second retry loop.

**Not claimed:** no `adapter opened` line. You may see
`found adapter but could not claim it: <code>` — capture the exact code, since
`ACCESS`, `BUSY` and `NOT_SUPPORTED` point at different causes.

You may equally see **nothing at all**. That is a known reporting gap, not a
crash: `TryOpen` bails early when `libusb_open_device_with_vid_pid` returns
NULL, and the warning lives after that call, on the claim failure. Silence
means "open failed", which is itself the useful signal. Confirmed on Windows.

Useful context either way:

```bash
system_profiler SPUSBDataType | grep -B 4 -A 12 -i '0x0337'
ioreg -p IOUSB -l -w 0 | grep -A 20 -i 'GameCube\|0x0337'
```

Dolphin drives this adapter on macOS, so a claim is believed possible — treat
that as a lead to chase in their USB backend, not as evidence this code is
right.

## Test 2 — if the claim fails, is the adapter still usable? (most important)

**Only meaningful if Test 1 failed.** If Test 1 succeeded, skip to Test 3.

The SDL ignore for `057e:0337` is registered unconditionally at startup,
before SDL initialises, whether or not an adapter is present or claimed. On
Windows that costs nothing, because without WinUSB the OS refuses to start the
device and SDL can't see it anyway. **On macOS the adapter is a perfectly good
HID device**, so if libusb can't claim it, that ignore takes away a gamepad
that would otherwise have worked.

Check: with the adapter connected and unclaimed, does it appear as an SDL
gamepad row in the input editor, and can you map and use a button?

- **Unusable** — nothing in the input editor, no input anywhere: this is a
  **macOS regression** introduced by the PR, and a merge blocker. Report it on
  #8 with the Test 1 log rather than fixing it blind; the real fix is to track
  claim state and apply or remove the ignore from the reader thread instead of
  deciding once at startup, which is a bigger change than it looks.
- **Usable as an SDL gamepad** — the fallback works and this is a non-issue.

Sanity check that the ignore is what's responsible, rather than something else,
by relaunching with `gControllers.GCAdapter.Enabled=0`. That returns before the
driver starts, so no ignore is ever registered and the adapter must show up
through SDL. If it appears then but not otherwise, the ignore is confirmed as
the cause.

## Test 3 — hotplug

Launch with the adapter **unplugged**, reach a point where you can see input,
then plug it in.

Expected: the ignore is registered at startup with nothing connected, and
`adapter opened` + `port N controller connected` follow when the adapter
appears. On macOS this should come from a hotplug event rather than the retry
loop (`hotplug=true`).

Watch for **doubled input** — one press registering twice, or a stick reading
double deflection. That would mean SDL opened the adapter alongside libusb.
Verified working on Windows; macOS is the platform where it could still go
wrong, since SDL can see the adapter there.

## Test 4 — things no platform has tested

None of this has been exercised anywhere yet:

- **In-game feel.** The driver assumes ±100 around centre for sticks and
  0..200 for triggers (`GCAdapter.cpp:23-24`). Check a full tilt reaches full
  N64 deflection and that smash vs. tilt thresholds feel right.
- **Rumble.** Needs the adapter's **second USB cable (5 V)** connected or it is
  a silent no-op in hardware. Confirm motors stop on quit — that is what
  `ShutdownGCAdapter()` in `PortShutdown` exists for.
- **Multiple controllers at once**, plus the per-port routing checkboxes.
- **Origin capture.** Hold a stick off-centre while plugging in; it should fall
  back to nominal centre rather than latching a bad origin (`kOriginSanity=40`).

## Known leftovers — not bugs to fix blind

- Per-mapping mutex and vector cost — the same pattern the existing SDL
  mappings use; left deliberately.
- No automated tests.
- `LIBUSB_ENABLE_UDEV OFF` in `src/CMakeLists.txt` was never confirmed to be a
  real libusb-cmake option. Irrelevant on macOS.
- The missing diagnostic described in Test 1 is a logging gap only. It was
  considered and deliberately left out of this PR as scope.
- The `ICON_FA_MOUSE_POINTER` change is an unrelated drive-by; a reviewer may
  ask for it to be split.

## Don't

- Don't commit the `.gitmodules` fork-URL edit.
- Don't force-push either `feature/gc-adapter` — both PRs are open.
- Don't add code to these PRs for problems that haven't been observed. Two
  fixes were already proposed on Windows for largely theoretical issues; one of
  them silently broke hotplug and had to be reverted. Observe first, then fix.
- Don't claim a platform is tested without a log line to back it. Both PR
  descriptions are written to separate what was observed from what was
  inferred — keep the macOS results honest the same way.
