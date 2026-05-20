# baltobu-dl-network-plugins

A small command-line tool that downloads the **network plugin shared
libraries** distributed by Bambu Lab for [Bambu Studio][bambu-studio]:

- `libbambu_networking` — the cloud / LAN networking layer
- `libBambuSource` — the camera / media source plugin
- `liblive555` — the RTSP/streaming stack
- The Agora runtime libraries that `libbambu_networking` links against

It talks to Bambu Lab's public resource API the same way the slicer
itself does (see `PresetUpdater::priv::sync_plugins` in the upstream
[BambuStudio][bambu-studio] source) and unpacks the published archive to
disk. By default it installs into the location Bambu Studio looks at on
the current host, so launching the slicer afterwards picks them up with
no further configuration.

This project is **not affiliated with or endorsed by Bambu Lab**. It is
an independent utility that fetches files from Bambu Lab's own public
CDN. The downloaded shared libraries themselves are owned and licensed
by Bambu Lab and its upstream vendors; this repository contains only
the fetch script.

## Why this exists

The slicer normally downloads these plugins itself on first launch via
its built-in updater. That works fine on developer machines but is
inconvenient when:

- you build Bambu Studio from source and want the plugins available
  before the first GUI run,
- you are setting up a fresh sandbox/container/CI image,
- you are cross-platform packaging and need the Mac/Windows binaries
  from a Linux box (or vice versa),
- you want to inspect or audit the binaries Bambu's updater would
  fetch.

## What each library does

A brief tour of what these binaries actually do inside Bambu Studio.
File paths below refer to the upstream [BambuStudio][bambu-studio]
source tree.

### `libbambu_networking` — control plane (cloud + LAN)

Loaded at startup by `NetworkAgent` (`src/slic3r/Utils/NetworkAgent.cpp`,
`initialize_network_module` → `dlopen`). It exposes ~80 C functions
prefixed `bambu_network_*` that the slicer wraps into a C++ `NetworkAgent`
object. Functionality includes:

- **Cloud account** — login / OAuth against Bambu Lab.
- **MQTT messaging** — long-lived pub/sub channel to Bambu's broker for
  telemetry and remote commands.
- **LAN discovery** — SSDP scan for printers on the local network.
- **Direct LAN connection** — talk to a printer over its local socket.
- **TLS / device cert handling**.

Without this library, Bambu Studio is effectively a local-only G-code
generator: no print upload, no printer status, no LAN or cloud features.

### `libBambuSource` — camera-stream media source plugin

Loaded by `wxMediaCtrl2` / `MediaPlayCtrl` and registered with the host
platform's media framework (GStreamer on Linux, Media Foundation on
Windows, AVFoundation on macOS). It claims the proprietary `bambu:///`
URL scheme, e.g.

```
bambu:///rtsps___<user>:<pass>@<lan-ip>/streaming/live/1?proto=rtsps
```

When the "live view" panel asks wxMediaCtrl to play one of those URLs,
the OS hands it to `libBambuSource`, which negotiates the printer's
RTSPS session, handles auth/decryption, and produces decoded video
frames for the UI.

### `liblive555` — RTSP/RTP/RTCP client

A pure dependency of `libBambuSource`. [LIVE555][live555] is a
well-known open-source C++ streaming-media library implementing the
protocol-level work: RTSP session setup, RTP packet handling, RTCP
feedback. `libBambuSource` sits on top of it.

### `libagora_rtc_sdk` + `libagora-fdkaac` — cloud-relayed camera

Runtime dependencies of `libbambu_networking` (not of `libBambuSource`).
[Agora][agora] is a commercial WebRTC-style real-time-comms SDK. This
path is used when the camera feed cannot go directly over LAN RTSP —
e.g. viewing the printer's camera from outside your network — and has
to be relayed through Bambu's cloud. The `fdkaac` shim is the
Fraunhofer AAC codec used for the audio side of that relay.

### Summary

| Library | Loaded by | Role | Talks to |
|---|---|---|---|
| `libbambu_networking` | `NetworkAgent` (`dlopen`) | Cloud + LAN control plane (auth, MQTT, discovery, commands) | `mqtt.bambulab.com`, the printer over LAN |
| `libBambuSource` | `wxMediaCtrl2` (registered with OS media framework) | Source plugin for the `bambu:///` URL scheme | The printer's RTSPS camera endpoint |
| `liblive555` | linked into `libBambuSource` | RTSP/RTP protocol client | Same — protocol details for camera stream |
| `libagora_rtc_sdk` + `fdkaac` | `dlopen`'d by `libbambu_networking` when needed | Cloud-relayed camera (off-LAN viewing) | Agora's WebRTC relay servers |

## Requirements

- `bash` 4+
- `curl`
- `unzip`
- `grep` with PCRE support (`grep -P`)

## Installation

Clone the repo and put the script on your `PATH`:

```sh
git clone <this-repo-url> baltobu-dl-network-plugins
ln -s "$PWD/baltobu-dl-network-plugins/baltobu-dl-network-plugins" ~/.local/bin/
```

Or just run it in place — it has no install step.

## Usage

```
baltobu-dl-network-plugins --platform <id> [--platform <id> ...]
                           [--version M.m.p.bb] [--dest <dir>]
                           [--country US|CN]
```

`--platform` is required; running with no arguments prints usage and
exits non-zero.

### Common invocations

Install Linux plugins into Bambu Studio's cache directory so the slicer
picks them up on next launch:

```sh
baltobu-dl-network-plugins --platform linux
# → ~/.config/BambuStudio/plugins/
```

Download every published platform into `./bambu_plugins/<os>/`:

```sh
baltobu-dl-network-plugins --platform all
```

Grab the macOS and Windows builds into a chosen directory:

```sh
baltobu-dl-network-plugins --platform macos --platform windows --dest ./out
```

### Flags

| Flag | Default | Meaning |
|------|---------|---------|
| `--platform <id>` | **required** | Repeatable. One of `linux`, `macos`, `windows`, `windows-arm`, or `all`. |
| `--version M.m.p.bb` | `02.07.00.00` (in-script) | Version sent to the API. The server returns the newest patch matching this major.minor. |
| `--dest <dir>` | Bambu cache when one platform; `./bambu_plugins/` otherwise | Where to extract. |
| `--country US\|CN` | `US` | Which API host to query (`api.bambulab.com` vs `api.bambulab.cn`). |
| `--help`, `-h` | — | Print usage and exit. |

### What gets downloaded

The API returns a zip per platform. Contents vary slightly:

| Platform | Archive layout |
|---|---|
| linux | five `*.so` files at the root |
| macos | three `*.dylib` files plus four `*.framework/` bundles (Agora) |
| windows | seven `*.dll` files |
| windows-arm | not currently published — script warns and skips |

### Bumping the default version

The script hardcodes a `DEFAULT_VERSION` so it can run without any
external context. When Bambu Studio's minor version moves (e.g.
`02.08.x.x`), edit the constant near the top of the script and commit.
Patch-level bumps within the same minor are handled automatically by
the server.

## How the protocol works

1. `GET https://api.bambulab.com/v1/iot-service/api/slicer/resource?slicer/plugins/cloud=<version>`
2. Send these headers (this is what selects the per-OS bundle):
   - `X-BBL-Client-Type: slicer`
   - `X-BBL-Client-Name: BambuStudio`
   - `X-BBL-OS-Type: linux | macos | windows | windows_arm`
3. The response is JSON containing a `resources[]` array; the entry of
   type `slicer/plugins/cloud` carries a `url` pointing at a zip on
   `public-cdn.bblmw.com`.
4. Download the zip and extract.

Nothing in this exchange is authenticated — these are public download
URLs the slicer hits on every fresh install.

## License

This script is licensed under the **GNU Affero General Public License
v3.0 or later**. See [`LICENSE`](./LICENSE) for the full text.

The shared libraries this tool *downloads* are proprietary works of
Bambu Lab and its upstream vendors and are subject to their own
license terms — this project does not grant you any rights to them.

> ⚠️ **Note on AGPL compliance:** As of 2026-05-18, the Software
> Freedom Conservancy has publicly alleged that distributing Bambu
> Studio bundled with these proprietary plugins violates the AGPLv3
> on the upstream PrusaSlicer/Slic3r code Bambu Studio inherits.
> See [`SFC-AGPL-FINDINGS.md`](./SFC-AGPL-FINDINGS.md) for a summary
> of the position and what it implies for users of this fetcher.

## Trademarks

"Bambu Lab", "Bambu Studio", and related marks are trademarks of
Shenzhen Bambu Lab Technology Co., Ltd. This project is independent and
is not produced, endorsed, sponsored, or supported by Bambu Lab.

[bambu-studio]: https://github.com/bambulab/BambuStudio
[live555]: http://www.live555.com/
[agora]: https://www.agora.io/
