# PM-241-BT macOS Client Driver

A macOS CUPS/PPD driver for the **PM-241-BT** thermal label printer, built to
be used as a **network client** against a PM-241-BT that is physically
connected (via USB) to a shared CUPS server.

## Why this driver exists

The vendor's stock PM-241-BT macOS driver crashes ("Filter failed" in CUPS,
printer status shows "Stopped on server") whenever it is used to print to a
**shared/networked** PM-241-BT — i.e. one plugged into a different machine
and accessed over the network via CUPS.

The root cause: the vendor driver bundles its own rendering filter
(`rastertolabeltspl`), which converts a print job all the way down to raw
TSPL printer-command language *locally on the client* before the job is even
sent. That's correct behavior for a printer plugged directly into your own
Mac. But when the same driver is installed on a client that's printing to a
**remote** CUPS queue, and that remote queue is running the *same* vendor
PPD/filter (because it needs it, since it's the machine actually attached to
the printer), the job gets converted to TSPL twice: once locally on the
client, and then a second time by the server, which chokes on data that's
already been converted and silently exits without printing anything useful.

This driver is the vendor's original PPD with the single line that causes
local TSPL rendering removed. Everything else — page sizes (including the
correct 4"x6" default), media/gap/darkness/print-speed options, and the
horizontal registration offset — is preserved, so the printer still shows up
correctly named with the right default paper size. The Mac now sends real
CUPS raster data over the network, and the server (which keeps its
unmodified, working copy of the vendor driver) does the one real conversion
to TSPL.

It also bakes in:
- A small (~2mm) imageable-area margin on every page size, so content
  isn't clipped by the printer's real (non-zero) printable edge.
- A default horizontal offset (`AdjustHoriaontal`) calibrated for our
  printer unit, to correct for its physical print-head registration.

## Is this driver for you?

This driver is **only** for printing to a PM-241-BT that's shared over the
network from another machine's CUPS install (e.g. a Linux box, Raspberry Pi,
NAS, etc. with the printer plugged into it via USB, and CUPS sharing it out).
It fixes the double-conversion crash described above, which only happens in
that networked/remote-queue scenario.

**If your PM-241-BT is plugged directly into your own Mac via USB**, don't
use this driver — install the vendor's original/stock driver instead. In
that setup there's no second machine to do the raster-to-TSPL conversion, so
your Mac needs to do it locally, which is exactly the step this driver
removes. Installing this driver for a directly-connected printer will result
in nothing printing at all (the job will just be raw CUPS raster with no
filter to turn it into something the printer understands).

Short version:
- PM-241-BT connected to *this* Mac via USB → use the vendor's stock driver.
- PM-241-BT connected to a *different* machine, shared via CUPS, and you're
  printing to it over the network → use this driver.

## Installation

**You must install this driver via the command line (`lpadmin`) — it cannot
be installed through the macOS "Add Printer" GUI.** This isn't a limitation
of this driver specifically: Apple removed the ability to point the Add
Printer dialog at an arbitrary local `.ppd` file in recent versions of
macOS. The GUI's "Select Software..." picker only shows drivers that are
already registered in `/Library/Printers/PPDs/Contents/Resources/` — it has
no "browse for a file" option anymore. `lpadmin` installs the driver
directly, without needing it pre-registered anywhere first.

### Steps

1. Download [`PM-241-BT-client.ppd`](./PM-241-BT-client.ppd) and
   [`PM-241-BT.icns`](./PM-241-BT.icns) from this repo and save them
   somewhere accessible, e.g. `~/Downloads/`.

2. Install the icon at the fixed path the PPD expects it at (this gives the
   printer its proper icon in Finder/Printers & Scanners instead of the
   generic printer icon):

   ```
   sudo mkdir -p /usr/local/share/PM-241-BT
   sudo cp ~/Downloads/PM-241-BT.icns /usr/local/share/PM-241-BT/PM-241-BT.icns
   ```

3. If a `PM-241-BT` printer already exists on your Mac (e.g. from a previous
   install attempt), remove it first:

   ```
   sudo lpadmin -x PM-241-BT
   ```

4. Install the printer using this driver, pointed at the CUPS server the
   PM-241-BT is physically connected to:

   ```
   sudo lpadmin -p PM-241-BT \
     -E \
     -v ipp://<cups-server-address>:631/printers/PM-241-BT \
     -P ~/Downloads/PM-241-BT-client.ppd \
     -D "PM-241-BT" \
     -L "<location>"
   ```

   Replace `<cups-server-address>` with the hostname/IP of your CUPS server,
   and `<location>` with whatever description you'd like (e.g. `Packpoint`).

5. Confirm it installed:

   ```
   lpstat -p PM-241-BT
   ```

   You should see the printer listed as idle. You'll also see a warning
   from `lpadmin` that says `Printer drivers are deprecated and will stop
   working in a future version of CUPS` — that's expected and harmless; it's
   CUPS's generic warning for any PPD-based (non-driverless) printer install,
   not specific to this driver.

6. Print a test label. It should now show up as `PM-241-BT` with its own
   icon (not the generic printer icon) in Printers & Scanners, default to
   the correct 4"x6" label size, and print without the "Filter failed"
   error.

   If the icon still shows generic, double check the icon file actually
   exists at `/usr/local/share/PM-241-BT/PM-241-BT.icns` (step 2) — macOS
   caches printer icons, so you may also need to remove and re-add the
   printer, or log out/in, for a newly-installed icon to show up.

### Server driver

[`PM-241-BT-server.ppd`](./PM-241-BT-server.ppd) is the vendor's original,
unmodified PPD, as installed on the CUPS server the printer is plugged into.
It keeps the `rastertolabeltspl` filter line that the client driver removes,
so the server does the single raster-to-TSPL conversion. It needs the
vendor's LabelPrinter package installed on the server, which provides that
filter.

```
sudo lpadmin -p PM-241-BT \
  -E \
  -v 'usb:///PM-241-BT?serial=<serial>' \
  -P PM-241-BT-server.ppd \
  -D "PM-241-BT" \
  -L "<location>" \
  -o printer-is-shared=true
```

`lpinfo -v` on the server lists the printer's USB URI, including its serial.

### If you need to fine-tune print alignment

Physical print-head registration can vary slightly between printer units. If
prints are consistently shifted toward one edge, you can adjust the default
horizontal offset by editing the `*DefaultAdjustHoriaontal` line in the PPD
(measured in mm) before installing, or by selecting "Horizontal Offset"
under this printer's options in the system Print dialog (System Print
dialog → printer options → Page Options group) for a one-off change.

Note that `lpoptions` (the CUPS command-line default-options tool) does
**not** affect print jobs sent from GUI apps on macOS — it only applies to
jobs printed via `lp`/`lpr`. Any option you want as a real global default for
GUI printing (Preview, browsers, etc.) needs to be baked into the PPD itself,
which is what this driver already does for the offset and margins.

## License

MIT — see [LICENSE](./LICENSE).
