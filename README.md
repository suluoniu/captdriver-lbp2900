# captdriver — Canon LBP2900 working on Linux + CUPS

A fork of [agalakhov/captdriver](https://github.com/agalakhov/captdriver) carrying
the fixes needed to make a **Canon LBP2900** actually print on Linux through CUPS,
instead of receiving a full job and silently sitting there.

> Tested on Raspberry Pi 4 / Ubuntu Server 24.04 (arm64) / CUPS 2.4.16 with a
> Canon LBP2900 over USB, shared to iOS, Android and HarmonyOS clients over IPP.
> Every page now comes out, and the job ends instead of hanging in
> "printing" forever.

---

## The problem

The LBP2900 is a GDI/CAPT printer: it has no PDL of its own, so a userspace filter
(`rastertocapt`) has to rasterise each page and speak Canon's CAPT protocol over
USB, including a back-channel handshake.

Upstream registers the LBP2900 with **conditional** extended-status queries:

```c
.get_status = capt_get_xstatus,      /* only reads 0xA0A8 when XSTATUS_CHNG is set */
.wait_ready = capt_wait_xready,
```

`XSTATUS_CHNG` is a bit in the *short* status word. **The LBP2900 never sets it.**
Consequences:

1. The extended status is never fetched, so `page_decoding` / `page_out` /
   `page_completed` stay frozen at stale values.
2. The end-of-page loops wait for those counters to converge — they never do, so
   the driver spins on `0xA0E0` (CHKSTATUS) polling **once per second, forever**.
3. Because `CAPT_FIRE` (0xE0A7) is only sent *after* that wait, the printer
   receives the entire page, decodes it, and then never fires the engine.

From the outside: the job goes to "printing", the printer does nothing, the queue
never drains, and the client keeps showing "printing" indefinitely.

## The fixes in this fork

### 1. Always-on extended status (the actual root cause)

The LBP3010 path does not rely on `XSTATUS_CHNG`, so the LBP2900 now uses it too:

```c
.get_status = capt_get_xstatus_only,
.wait_ready = capt_wait_xready_only,
```

This single change is what makes pages print at all.

### 2. Relative-increment page completion detection

Even with the counters live, this particular unit **still only settles
`page_out` / `page_completed` after the job ends** — they do not move while the
job is running (verified: 12 s of polling after `CAPT_FIRE`, `pages 3/2/2/2`
unchanged, while the sheet was physically coming out). So the upstream equality
test

```c
if (status->page_out == status->page_decoding)
```

can never succeed here. This fork snapshots each counter at the start of a page
and treats *"advanced past the baseline"* as completion:

```c
lbp_base_out = status->page_out;
...
if (status->page_out > lbp_base_out ||
    status->page_out == status->page_decoding)
        return true;
```

The original equality test is kept as an alternative branch, so units where it
does work are unaffected.

### 3. Poll interval 1000 ms → 100 ms

`capt_wait_ready()` and friends used `sleep(1)`, adding up to a full second of
dead time per handshake step. Now `usleep(CAPT_POLL_US)` (100 ms), via
`CAPT_POLL_US` in `src/std.h`.

### 4. Safety-net timeouts on every wait loop

Previously each loop was `while (1)`. A freak page whose status never settles
would hang the whole queue. All loops now have an upper bound
(12 s per page, 4 s for the job epilogue) and log a warning instead of spinning.

### 5. Printer state reported to CUPS

`STATE: -media-empty` on page ejection and at job start, so a stale
"out of paper" reason does not linger in the client's queue window.

---

## Measured result

Single-page text job, Raspberry Pi 4, same hardware and file:

| | before | after |
|---|---|---|
| Time to job completion | never (queue had to be cancelled) | **~19.5 s** |
| With only the upstream+macOS patch, before the counter fix | 65 s+ per page (all three wait loops timing out) | — |

Of the 19.5 s: ~2 s rendering, ~1 s USB transfer, ~12 s waiting for the LBP2900's
mechanical first-page time, ~4 s epilogue. The 12 s is the engine, not the
software — compressing it further risks leaving a partial page stuck in the
printer's memory (which requires a power cycle to clear).

---

## Build

Dependencies (Debian/Ubuntu):

```sh
sudo apt install build-essential libcups2-dev libcupsimage2-dev
```

Compile:

```sh
cd src
gcc -O2 -Wall -o rastertocapt *.c $(cups-config --cflags) $(cups-config --image --libs)
sudo install -m 755 rastertocapt /usr/lib/cups/filter/rastertocapt
sudo systemctl restart cups
```

> Note: the flag is `cups-config --image --libs`, **not** `--image-libs`.
> The latter is not a valid option and gets passed to gcc as a filename.

Autotools builds (`./autogen.sh && ./configure && make`) work as upstream.

## Install the queue

```sh
sudo lpadmin -p LBP2900 -E -v usb://Canon/LBP2900 \
     -P /path/to/Canon-LBP2900.ppd -o printer-is-shared=true
sudo cupsenable LBP2900 && sudo cupsaccept LBP2900
```

The PPDs shipped in the repo root (`Canon-LBP2900.ppd`, `Canon-LBP3000.ppd`) are
the ones this was tested with.

---

## Troubleshooting notes from the field

These cost the most time, so they are written down:

**`quirks` files are loaded by directory, not by name.** CUPS' USB backend
`load_quirks()` reads *every* regular file in `/usr/share/cups/usb/`. If you leave
a backup like `org.cups.usb-quirks.bak` there containing
`0x04a9 0x2676 unidir`, that line is still loaded and kills the back-channel —
producing `CAPT: no reply from printer`. Keep backups outside that directory.

**Back-channel desync after a cancelled job.** The printer emits a 56-byte status
burst on connect; if a previous job died mid-handshake, the next job's first
6-byte read lands mid-stream and fails with
`bad reply from printer, expected A1 A1 xx xx xx xx`. It clears itself once the
burst is consumed.

**`usblp` must not hold the device.** Blacklist it (`/etc/modprobe.d/`) so the
CUPS libusb backend has exclusive access.

**cupsd exiting under you.** Ubuntu ships CUPS with `IdleExitTimeout 60` plus
socket activation, so `cupsd` exits when idle and the next client pays a cold
start. Set `IdleExitTimeout 0` on a print server.

---

## Credits

- [agalakhov/captdriver](https://github.com/agalakhov/captdriver) — the driver
  this is forked from.
- [duy12i1i7/canon-LBP2900-for-macOS](https://github.com/duy12i1i7/canon-LBP2900-for-macOS)
  — their `patches/lbp2900-macos.patch` identified the conditional-status root
  cause and supplied the `capt_get_xstatus_only` strategy, the 100 ms poll
  interval and the `PAGE:`/`STATE:` reporting used here.
- [mounaiban/captdriver](https://github.com/mounaiban/captdriver) — the more
  actively maintained upstream.

`patches/captdriver-lbp2900-linux.patch` contains the complete diff against
upstream master, if you would rather apply it than clone this fork.

## License

GPL-2.0, see `COPYING`. Original code © the captdriver authors (see `AUTHORS`).
