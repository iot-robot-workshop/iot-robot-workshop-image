FROM mcr.microsoft.com/devcontainers/python:3.12-bookworm

RUN apt-get update \
 && apt-get install -y --no-install-recommends git usbutils \
 && rm -rf /var/lib/apt/lists/*

# Add the '99-platformio-udev.rules' file to suppress the warning during upload. This file isn't necessary because the
# 'serial-perms.sh' file already handles the port permissions
RUN mkdir -p /etc/udev/rules.d \
 && curl -fsSL https://raw.githubusercontent.com/platformio/platformio-core/develop/platformio/assets/system/99-platformio-udev.rules \
      -o /etc/udev/rules.d/99-platformio-udev.rules

USER vscode
ENV PLATFORMIO_CORE_DIR=/home/vscode/.platformio

# Install PlatformIO Core where the VS Code extension looks for it (~/.platformio/penv),
# so the extension does not download its own copy on first start.
# Pin a version for reproducible images, e.g. PIO_SPEC="platformio==<version>".
ARG PIO_SPEC=platformio
RUN python -m venv $PLATFORMIO_CORE_DIR/penv \
 && $PLATFORMIO_CORE_DIR/penv/bin/pip install --no-cache-dir "$PIO_SPEC"
ENV PATH=$PLATFORMIO_CORE_DIR/penv/bin:$PATH

# Pre-bake platform, toolchain and framework by building a throwaway project.
COPY --chown=vscode:vscode prebake /tmp/prebake
RUN cd /tmp/prebake \
 && pio run \
 # Optional: bake global libraries too, e.g.
 # && pio pkg install -g -l "bblanchon/ArduinoJson@^7" \
 # Download filesystem image tools. Needed when uploading target
 && pio pkg install -g \
      -t "platformio/tool-mkspiffs@~2.230.0" \
      -t "platformio/tool-mklittlefs@~1.203.0" \
      -t "platformio/tool-mkfatfs@~2.0.0" \
 && rm -rf /tmp/prebake
