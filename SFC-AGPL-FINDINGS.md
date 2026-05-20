# SFC findings on Bambu Studio AGPLv3 violations

> ⚠️ This document summarizes a third-party legal position. It is not
> legal advice. For anything actionable, consult a lawyer.

This repository is a fetcher for the proprietary network plugin
libraries Bambu Studio loads at runtime. As of 2026-05-18 those
plugins are the subject of a public enforcement statement by the
**Software Freedom Conservancy (SFC)**, alleging that their use
violates the AGPLv3 license under which Bambu Studio is distributed.

This file records what SFC said, the relevant code patterns in
upstream Bambu Studio, and the implications for users of this
fetcher.

## Source

- **Article:** "Bambu Studio 3D Printer AGPL Violation Response",
  Software Freedom Conservancy, 2026-05-18.
- **URL:** <https://sfconservancy.org/news/2026/may/18/bambu-studio-3d-printer-agpl-violation-response/>

## What SFC alleges

### Violation 1 — failure to provide Corresponding Source

Bambu Studio is licensed under AGPLv3 (`LICENSE` in the upstream
[BambuStudio][bambu-studio] repository is the canonical AGPLv3 text;
`README.md` declares the project AGPL-3.0). Significant portions of
the codebase are inherited from PrusaSlicer and Slic3r and are
copyrighted by Prusa Research, Alessandro Ranellucci, and other
upstream contributors — also under AGPLv3.

The slicer dynamically loads three proprietary shared libraries at
runtime:

- `libbambu_networking.so` / `bambu_networking.dll` / `libbambu_networking.dylib`
- `libBambuSource.so` / `BambuSource.dll` / `libBambuSource.dylib`
- `liblive555.so` / `live555.dll` / `liblive555.dylib` (LGPL upstream,
  but distributed without source by Bambu)

…plus Agora SDK runtime dependencies. The dynamic-load pattern is in
`src/slic3r/Utils/NetworkAgent.cpp`: `dlopen` (line ~257) followed by
~80 `dlsym` lookups (lines 284–308+) binding `bambu_network_*`
symbols that the slicer then calls synchronously inside the AGPL
process.

SFC argues this triggers AGPLv3 §1's definition of Corresponding
Source, which they quote:

> *"such as by intimate data communication or control flow between
> those subprograms and other parts of the work."*

And concludes:

> *"Object Code combined with AGPLv3'd software must also be licensed
> under AGPLv3. … As such, these Object Code libraries are governed
> by the AGPLv3."*

Bambu does not publish source for the proprietary plugins, so under
SFC's reading, the combination distributed today is a license
violation against the upstream AGPL contributors whose code Bambu
inherited.

### Violation 2 — cease-and-desist against an AGPL-permitted fork

Bambu Lab issued cease-and-desist demands against **Paweł Jarczak**,
who maintained an OrcaSlicer fork that allowed users to operate
Bambu printers without the proprietary libraries. SFC reads this as
a direct violation of AGPLv3 §10¶3:

> *"You may not impose any further restrictions on the exercise of
> the rights granted or affirmed under this License."*

Forking and modifying AGPL software (including stripping out
proprietary dependencies) is exactly what the license permits;
attempting to use trademark or contract claims to suppress that
exercise is the conduct §10¶3 prohibits.

## What SFC is doing about it

The response is operational, not just rhetorical:

- Reverse-engineering the proprietary libraries to produce **AGPL-licensed
  drop-in replacements**, asserting:

  > *"SFC and our volunteers are within our rights to reverse-engineer
  > these libraries for the purpose of creating our own Source Code
  > that can function as a drop-in replacement in Bambu Studio."*

- **Maintaining an OrcaSlicer fork** that operates Bambu printers
  without the proprietary components.
- Building a clean-slate replacement slicer (codename **"viscose"**)
  integrating the reverse-engineering work.
- Standing up a permanent **committee for 3D-printer software freedom**
  to monitor for further violations.

## Where this stands legally

- **No court has ruled on this specific case.** SFC's position
  reflects the long-standing FSF interpretation of GPL/AGPL
  derivative-work scope, but the question of whether dynamic linking
  via `dlopen`+`dlsym` creates a derivative work for license purposes
  has very little case law in any jurisdiction.
- What changed on 2026-05-18 is that a credible enforcement
  organization is now publicly pressing this view *against this
  specific product*, with active remediation work underway. That
  moves the situation from "FSF opinion + silence" to "active
  enforcement targeting Bambu Studio in particular."
- The proprietary libraries are not legally "converted" to AGPL by
  being loaded into AGPL code — a license cannot be unilaterally
  imposed on a third party's copyright. SFC's claim is that
  *distributing the combination* (slicer + plugin) without
  AGPL-compatible source for the plugin is what violates the AGPL on
  the upstream code Bambu inherited.

## Implications for users of this fetcher

- **The fetcher itself (`baltobu-dl-network-plugins`) is unaffected.**
  This repo ships only the bash script; it does not contain or
  redistribute the binaries. Its AGPLv3 license stands.
- **Running the downloaded binaries on your own machine** is the same
  thing the slicer's own updater does on first launch. SFC's claim
  is against distribution, not personal use.
- **Redistributing or mirroring the binaries yourself** carries
  meaningfully more risk after 2026-05-18 than before. If you bundle
  the slicer plus these plugins into a package you publish, SFC's
  reading is that you are the redistributor responsible for AGPL
  compliance — including providing AGPL-licensed Corresponding
  Source for the plugins, which you cannot do without Bambu's
  cooperation.
- **Watch the SFC replacement work.** If "viscose" / the OrcaSlicer
  fork ships AGPL-licensed equivalents of `libbambu_networking` and
  friends, that becomes a clean path to running a Bambu printer from
  a fully-free slicer, and this fetcher's audience shrinks
  accordingly.

## See also

- [`README.md`](./README.md) — what this tool does and the libraries
  it downloads.
- [`LICENSE`](./LICENSE) — AGPL-3.0-or-later, covering the fetcher
  itself (not the binaries it downloads).

[bambu-studio]: https://github.com/bambulab/BambuStudio
