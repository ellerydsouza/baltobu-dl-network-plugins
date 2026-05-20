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

Bambu Lab issued cease-and-desist demands against **Paweł Jarczak**.
SFC describes what he built:

> *"A software developer and Bambu Lab user (Paweł Jarczak) released
> another mechanism to integrate with Bambu Studio's server side
> components that did not require replacing or modifying the
> dynamically linked libraries. Instead, Paweł made changes to a
> different AGPLv3'd slicer (Orca Slicer) by merely examining the
> (incomplete) source code for Bambu Studio. Those Orca Slicer
> modifications allowed users to replace Bambu Studio and instead
> combine Orca Slicer via intimate data communication with Bambu
> Studio's currently-source-unavailable parts that run on Bambu
> Lab's servers."*

Critically, Paweł's fork **did not use `libbambu_networking.so` at
all**. Instead, he reverse-engineered the wire protocol that
BambuNetwork uses to talk to Bambu Lab's cloud servers, and
reimplemented that conversation directly inside OrcaSlicer — a
clean-room reimplementation of the *protocol*, not a redistribution
of any proprietary binary.

Bambu's response:

> *"Bambu demanded that Paweł remove the fork of OrcaSlicer with
> these changes from Github."*

Paweł complied (under protest) and the original repository at
`github.com/jarczakpawel/OrcaSlicer-bambulab` is no longer
available. SFC reads the C&D as a direct violation of AGPLv3 §10¶3:

> *"You may not impose any further restrictions on the exercise of
> the rights granted or affirmed under this License."*

The §10¶3 argument is sharp here precisely *because* Paweł's work
was a clean-room reimplementation of the network protocol. He was
modifying AGPL-licensed Orca Slicer (which inherits from AGPL Bambu
Studio, which inherits from AGPL PrusaSlicer / Slic3r) — exactly
what AGPL §2 grants him the right to do. Bambu used trademark
and/or ToS claims to suppress that AGPL-permitted modification,
which is the conduct §10¶3 prohibits.

> ⚠️ A separate repository named `OrcaSlicer-bambulab` exists on
> GitHub re-uploaded by a third party ("Jake") after Paweł's
> removal. That repository **restores BambuNetwork support** —
> i.e., it is the opposite of what Paweł built. Do not confuse the
> two: the name collision is misleading.

## What SFC is doing about it

The response is operational, not just rhetorical:

- Reverse-engineering the proprietary libraries to produce **AGPL-licensed
  drop-in replacements**, asserting:

  > *"SFC and our volunteers are within our rights to reverse-engineer
  > these libraries for the purpose of creating our own Source Code
  > that can function as a drop-in replacement in Bambu Studio."*

- **Continuing Paweł's OrcaSlicer fork**, hosted on SFC's own
  forge under the `baltobu` namespace:
  <https://f.sfconservancy.org/baltobu/orca-slicer-for-bambu>.
  The article describes it as a continuation that "will build on
  Paweł's work" — i.e., the protocol-reimplementation approach, not
  a re-integration of `libbambu_networking.so`.
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
