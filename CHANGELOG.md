# Changelog

## [Unreleased]

### Added

- The applet sweep now requires a man page for every announced name, on each
  target it sweeps. A name the user can run and cannot read about is a
  half-shipped program, and it failed quietly: `unpin man <pkg> <name>` found
  nothing and the listing showed the name with an empty description.

  Per-target because that is the shape the defects took — binutils' `.exe` once
  shipped all 18 pages under `x86_64-w64-mingw32-*` so not one announced name
  resolved, dosfstools' `.exe` missed 7 of 10, lz4's 2 of 3. Each of those is a
  page that exists on another target.

  Two new manifest fields carry the exceptions, both declared in the caller's
  flake (nix-lib 9477c7a or newer): `applets_no_man`, names upstream documents
  nowhere; and `applets_man_page`, the single page that documents every applet
  (busybox describes all 396 in `busybox.1`). Neither is a blanket waiver — a
  name in `applets_no_man` that *does* carry a page here fails too, and a
  declared `applets_man_page` must itself be embedded.

  A caller whose nix-lib predates the fields reads them as empty, which is the
  strict end of the range, not a silent skip. Measured before switching on:
  every announced name in the 475 published payloads of the 101 packages not
  touched by this round already resolves.
