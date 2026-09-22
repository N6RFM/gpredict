# Alpha-5 Catalog Number Support (branch: alpha5-on-v2.3.115)

## What this branch is

This branch adds Alpha-5 catalog number decoding on top of the exact
commit (`0f3beb6`) that Ubuntu's `gpredict` 2.3-115 apt package was built
from — not on top of the `master` branch of this fork.

## Why a separate base instead of `master`

This fork's `master` branch (and the `fix-alpha5-space-padded-catnr`
branch) contains a rotator-control display bug: the "Read" fields in the
Rotator Control window never update, even though the underlying rotctld
connection and hardware communication work correctly. The bug was
confirmed to exist even before any Alpha-5 changes were made (verified by
checking out the commit immediately prior to the Alpha-5 work and testing
independently), and also reproduces on current upstream `csete/gpredict`
master. It is therefore an upstream regression unrelated to the Alpha-5
work, still unidentified as of this writing.

Since the Ubuntu-packaged 2.3-115 release does NOT have this bug, this
branch starts from that exact commit and adds only the Alpha-5 decoding
logic, to get a build with both working rotator control and NORAD ID
support >= 100000.

## What changed

CelesTrak's SATCAT crossed 100000 on 2026-07-11. Objects cataloged past
that point use Alpha-5 encoding in the TLE catalog number field: the
leading digit is replaced with a letter (A-Z, excluding I/O) to represent
values up to 339999 in the existing 5-character field width.

Gpredict parsed this field with plain numeric conversion in two places,
so any Alpha-5-encoded catalog number silently decoded to 0, causing
those satellites to collide/fail to load correctly.

### Files changed
- `src/sgpsdp/sgp_in.c` — added `decode_alpha5_catnr()`, used in
  `Convert_Satellite_Data()` in place of the old `atoi()` call.
- `src/tle-update.c` — added the same decoder, used in `read_fresh_tle()`
  in place of the old `g_ascii_strtod()` call.

Both decoders handle:
- Legacy zero-padded numeric catalog numbers (e.g. `00900`)
- Legacy space-padded numeric catalog numbers (e.g. ` 1234`)
- Alpha-5 encoded numbers (e.g. `A0057` -> 100057)

### Verification
- Confirmed against Space-Track's published Alpha-5 test values
  (A0000->100000 ... Z9999->339999).
- Confirmed end-to-end in Gpredict: loaded a CelesTrak TLE file
  containing Alpha-5 satellites (e.g. `SOYUZ-MS 29`, catalog field
  `A0057`), and the Satellite Info panel correctly displays
  "Catalogue number: 100057".
- Confirmed rotator control (Engage / Read fields) works correctly on
  this branch, unlike `master`/`fix-alpha5-space-padded-catnr`.

## Known limitation

The one line from the original `fix-alpha5-space-padded-catnr` branch
that unconditionally overwrote `sat->tle.catnr` with the filename-derived
catalog number (in `gtk-sat-data.c`) was NOT carried over to this branch.
Testing showed it did not fix or affect the rotator display bug, and
since the decoder in `sgp_in.c` now handles Alpha-5 correctly at the
source, the overwrite is likely unnecessary. Revisit if any catalog
number mismatches are observed.
