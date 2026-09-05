# Changelog

## [Unreleased]

### Fixed

- The applet sweep no longer fails on a package that embeds no man page at all.
  Reading the page names out of the payload produced `$null` rather than an
  empty list when there were none, and writing that out threw. svt-av1, which
  documents nothing, was the first package to hit it.

### Added

- A GUI-subsystem `.exe` can now be smoked. `smokeWindows = false` says the
  binary has no console and so no stdout to grep, and until now that meant CI
  built it, unpacked it, checked its PE header — and never ran it. gvim.exe
  shipped for months with an embedded runtime tree that opened files by exact
  name and returned nothing to any glob; no step in this workflow could have
  noticed.

  The new `smoke_windows_gui` manifest field (nix-lib `smokeWindowsGui`) says
  how to make the binary answer in a file instead: `files` are written next to
  the `.exe`, `args` run it, and `outFile` must exist afterwards and match
  `pattern`.

  The probe files are written with LF endings — git-bash's jq emits CRLF,
  and a script the .exe interprets then carries a `\r` on every line.
  Measured on the runner: gvim read `qa!\r`, which is not a command, so it
  never quit and the step hit its limit with nothing written.

  The step waits for the FILE, not for the process. The first run on
  windows-2022 timed out with gvim.exe still alive after 120 s, and waiting
  on the process could not say whether it had done its work: a windowed
  program on a runner with no interactive desktop may write everything it
  was asked to and still not tear down. What the smoke asks is whether the
  binary ran and reached its payload, so the file is the answer — the poll
  ends the moment it appears, a process still alive at the end is terminated
  and reported, and a run that writes nothing or exits nonzero fails.

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
