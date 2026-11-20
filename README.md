# IoT-Robot Workshop Dev Container Image

Builds and publishes the Docker image used by the IoT-Robot workshop dev container
(VS Code + PlatformIO + ESP-IDF). The image contains PlatformIO Core, the ESP32
platform, the ESP-IDF framework and the toolchain, so students do not have to
download several GB on their first build.
 
This repo is for the workshop maintainer only. Students use the separate
**student template repo**, which references the image published from here.
 
## How the two repos fit together
 
```
this repo (image)                    student template repo
-----------------                    ---------------------
image/Dockerfile        --build-->   .devcontainer/devcontainer.json
image/prebake/                        references ghcr.io/<owner>/<image-name>:<tag>
.github/workflows/image.yml          platformio.ini, src/, scripts/attach.ps1
        |
        v
ghcr.io/<owner>/<image-name>:<tag>
```
 
The only coupling between the repos is the toolchain configuration (see
[Keeping the repos in sync](#keeping-the-repos-in-sync)).
 
## Repository layout
 
```
.
├── image/
│   ├── Dockerfile                # image definition
│   └── prebake/                  # throwaway project used only at image build time
│       ├── platformio.ini
│       └── src/main.c
├── .github/workflows/image.yml   # builds and pushes the image to GHCR
└── README.md
```
 
### What is baked into the image
 
- Python 3.12 (Debian bookworm, from the Dev Containers base image), git, usbutils
- PlatformIO Core in `~/.platformio/penv`, where the VS Code PlatformIO extension looks for it
- The platform, framework and toolchain downloaded by building `image/prebake`
  (ESP32 platform, ESP-IDF, CMake, Ninja, esptool, compiler toolchain)
- Filesystem tools that PlatformIO otherwise fetches at the first upload:
  `tool-mkspiffs`, `tool-mklittlefs`, `tool-mkfatfs`
- PlatformIO udev rules file (only silences a warning; serial permissions are
  handled by the template's `serial-perms.sh`)

### What the prebake project is
 
`image/prebake/` is not a student project. It is a minimal project that the
Dockerfile builds once with `pio run` so PlatformIO downloads everything into the
image. It is deleted at the end of the build and never appears in the
students' workspace.
 
## Configuration
 
Decide on the image name once and use it everywhere:
 
| Where | What to set |
|---|---|
| `.github/workflows/image.yml` | the two `tags:` lines: `ghcr.io/<owner>/<image-name>:<tag>` and `:latest` (owner must be lowercase) |
| Template repo `devcontainer.json` | `"image": "ghcr.io/<owner>/<image-name>:<tag>"` |
| `image/prebake/platformio.ini` | `platform`, `board`, `framework` (see below) |
| `Dockerfile`, optional | `PIO_SPEC` build arg to pin PlatformIO Core, e.g. `platformio==<version>` |
 
## Keeping the repos in sync
 
The `platform`, `board` and `framework` lines in `image/prebake/platformio.ini`
must be identical to those in the student template's root `platformio.ini`, with
the platform version pinned, for example:
 
```ini
platform = espressif32@~8.0.0
board = esp32dev
framework = espidf
```
 
If they differ, the baked packages are not used and PlatformIO downloads what it
needs at the first build. It still works, but slowly.
 
## Build and test locally
 
From the repo root, with Docker running:
 
```bash
docker build -t <image-name>:dev image
```
 
The first build takes a while because it downloads the full ESP-IDF toolchain.
Check the result:
 
```bash
docker run --rm -it <image-name>:dev bash -c "pio --version && ls ~/.platformio/packages"
```
 
You should see the toolchain, `framework-espidf`, CMake and Ninja. To test with a
board, set `"image": "<image-name>:dev"` in a checkout of the template repo,
run `scripts/attach.ps1` on Windows, and use **Dev Containers: Reopen in Container**.
On an ARM machine add `--platform linux/amd64`, since student PCs are x86.
 
## Publish
 
**Via GitHub Actions (default).** Pushing to `main` with changes under `image/`
(or to the workflow file) builds and pushes the image. You can also start it from
the Actions tab (`workflow_dispatch`).
 
After the first successful run, set the package to **Public**
(Package settings, then Change visibility), otherwise students cannot pull it
without logging in. For an organization, public packages may first have to be
allowed in the org settings.
 
**Manually.**
 
```bash
echo <TOKEN> | docker login ghcr.io -u <github-user> --password-stdin
docker tag <image-name>:dev ghcr.io/<owner>/<image-name>:<tag>
docker push ghcr.io/<owner>/<image-name>:<tag>
```
 
The token is a personal access token with the `write:packages` scope.
 
## Updating the image
 
1. Edit the files in `image/` (new platform pin, extra baked tools or libraries, ...).
2. Build and test locally (see above).
3. Bump the image tag (for example `1` to `2`) in `image.yml`, push, and wait for a green run.
4. In the template repo:
   - update the pinned platform in `platformio.ini` if it changed,
   - set the new tag in `devcontainer.json`,
   - bump the volume name `pio-core-v1` to `pio-core-v2`. The volume is seeded from the
     image only when it is first created, so without a new name students keep the old toolchain.
5. Test on one clean machine, with a fresh clone, before the next workshop.
### Baking additional packages
 
If PlatformIO prints `Tool Manager: Installing ...` during a student's first build
or upload, that package was not baked. Add it to the Dockerfile with the same
name and version spec, for example:
 
```dockerfile
&& pio pkg install -g -t "platformio/<tool-name>@<version-spec>"
```
 
Global libraries use `pio pkg install -g -l "<library>"`. ESP-IDF components belong
in the student project (`components/` or `idf_component.yml`), not in the image.
 
## Troubleshooting
 
| Symptom | Likely cause |
|---|---|
| `COPY prebake ... not found` during build | `prebake/` is not next to the Dockerfile, or the build context is wrong. Build with `docker build -t <name> image`. |
| `unauthorized` when pulling from GHCR | Image or tag does not exist under that name, package is still private, or a stale `docker login`. Try `docker logout ghcr.io`. |
| Students' first build downloads a lot | Platform, board or framework in the template differ from `prebake`, or the pin differs. |
| `KeyError: 'version'` from `espidf.py` | Generic symptom of a failed CMake configure. Delete `.pio` and run `pio run -v`; read the `CMake Error` lines before the traceback. |
| Port not visible in the container | The board is not attached on the Windows side (`attach.ps1`), or the cable has no data lines. |
 
## Requirements for students (documented in the template repo)
 
Docker Desktop with the WSL2 backend, VS Code with the Dev Containers and WSL
extensions, and `usbipd-win` (`winget install usbipd`).
 