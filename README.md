# IDA Pro Docker Release Pipeline

Automated multi-arch (`linux/amd64` and `linux/arm64`) Docker build and release pipeline for IDA Pro.

## Published Images

Images are published to GHCR: `ghcr.io/theunknownzoo/ida-pro`

| Variant | Tag | Description |
|---|---|---|
| **Headless** | `:9.4` (or `:latest`) | Headless CLI runtime (`idat`, `idalib`), IDAPython, Node.js 26 LTS, and uv. No X11/GL dependencies. |
| **GUI** | `:9.4-gui` (or `:latest-gui`, `:gui`) | Includes X11 and OpenGL libraries needed to run the `ida` desktop GUI over an X server. |

All tags are multi-arch manifests supporting both `linux/amd64` and `linux/arm64` natively.

## Triggering a Release

Run the workflow manually via GitHub Actions:

1. Go to **Actions** → **Docker**.
2. Click **Run workflow**.
3. Configure the inputs:
   - **`tag`**: Target version tag (e.g. `9.4`).
   - **`source_ref`**: Branch, tag, or commit SHA in the private source repo (default: `main`).
   - **`push_latest`**: Also tag manifests as `latest` and `latest-gui` (default: `false`).

## Quick Usage

### Headless (`idat` / idalib)

```bash
docker run --rm -it \
  -v "$PWD/work:/work" \
  -v ida-state:/home/ninja/.idapro \
  ghcr.io/theunknownzoo/ida-pro:9.4 idat -A -B /work/sample.bin
```

### GUI (via X11)

```bash
docker run --rm -it \
  -e DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v ida-state:/home/ninja/.idapro \
  ghcr.io/theunknownzoo/ida-pro:9.4-gui ida /work/sample.bin
```

> **Note**: Mount a valid, node-locked license (`*.hexlic`) into `/home/ninja/.idapro` at runtime. The images contain no cracked binaries, patches, or licenses.

## Workflow Architecture

- **Native runners**: Builds run on `ubuntu-24.04` (amd64) and `ubuntu-24.04-arm` (arm64) to avoid QEMU emulation overhead.
- **Fast parallel downloads**: Each runner fetches only its architecture's installer using segmented HTTP Range requests.
- **Layer caching**: Headless and GUI variants share the exact same unpacked IDA installation stage; the GUI variant reuses the layer cache and only adds the Debian GUI packages.
- **Flattened GUI tags**: the GUI variant is built from the Dockerfile `squash` stage, so its rootfs is a single layer (`docker history` shows one `COPY`, and the chown pass no longer stores the install twice). Headless keeps per-step layers.
- **Privacy**: The source repository stays private. Docker build logs are piped to a file and only surfaced if an error occurs.
