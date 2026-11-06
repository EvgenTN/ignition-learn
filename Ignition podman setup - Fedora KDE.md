# Ignition + MariaDB with Podman (Fedora KDE 45, learning)

This is the Fedora KDE (Plasma, Wayland) variant of the setup guide. The Gateway and MariaDB parts are identical to the GNOME version; only the **Designer** part (section 6) differs.

**Tested on:** Fedora KDE 45, Plasma Wayland session, Ignition 8.3, Designer Launcher 1.3.10 (bundled Zulu OpenJDK 17).

---

## 🟢 DAILY USE (quick reference)

**Start the Gateway + MariaDB:**
```bash
podman pod start ignition-stack
```
Check: http://localhost:8088 (login: `admin` / `ignition`)

**Stop the Gateway + MariaDB** (your data is preserved):
```bash
podman pod stop ignition-stack
```

**Check what is currently running:**
```bash
podman pod ps
podman ps --pod
```

**Open Designer:**
```bash
toolbox enter ignition-designer
```
```bash
cd ~/ignition-designer/designerlauncher/designerlauncher
env -u WAYLAND_DISPLAY mesa_glthread=false _JAVA_AWT_WM_NONREPARENTING=1 \
  JAVA_TOOL_OPTIONS="-Dawt.robot.screenshotMethod=x11 -Dsun.java2d.xrender=false" \
  ./designerlauncher.sh
```
Run the launcher in the foreground. After closing Designer, run `exit` to leave the toolbox.

> The path above has a nested `designerlauncher/designerlauncher` because of how the archive unpacked on the tested machine. The correct directory is the one that contains `designerlauncher.sh` (check with `ls`).

Optional: put the launch command into a wrapper script (inside the toolbox) so you don't have to retype it:
```bash
cat > ~/ignition-designer/start-designer.sh <<'EOF'
#!/bin/bash
cd ~/ignition-designer/designerlauncher/designerlauncher || exit 1
exec env -u WAYLAND_DISPLAY mesa_glthread=false _JAVA_AWT_WM_NONREPARENTING=1 \
  JAVA_TOOL_OPTIONS="-Dawt.robot.screenshotMethod=x11 -Dsun.java2d.xrender=false" \
  ./designerlauncher.sh
EOF
chmod +x ~/ignition-designer/start-designer.sh
```
Then: `toolbox enter ignition-designer` and `~/ignition-designer/start-designer.sh`.

> Do not put these variables into `~/.bashrc`: the home directory is shared with the host, so they would affect every application on the host.

**Open a MariaDB console (for troubleshooting):**
```bash
podman exec -it mariadb mariadb -u ignition -pignition ignition_learning
```

---

## ⚙️ INSTALLATION (one-time setup)

### 1. Create volumes for the Gateway and MariaDB

Volumes preserve data (projects, tags, and tables) if you remove and recreate a container.

```bash
podman volume create ignition-data
podman volume create mariadb-data
```

### 2. Create a pod (to manage the Gateway and MariaDB together)

Ports are published on the pod. Containers in the pod share a network namespace and can reach each other through `localhost`.

```bash
podman pod create --name ignition-stack \
  -p 8088:8088 \
  -p 8043:8043 \
  -p 3306:3306
```

### 3. Start MariaDB

```bash
podman run -d --name mariadb \
  --pod ignition-stack \
  -e MARIADB_ROOT_PASSWORD=ignition \
  -e MARIADB_DATABASE=ignition_learning \
  -e MARIADB_USER=ignition \
  -e MARIADB_PASSWORD=ignition \
  -v mariadb-data:/var/lib/mysql \
  docker.io/library/mariadb:11
```

### 4. Start the Ignition Gateway (8.3)

```bash
podman run -d --name ignition-gw \
  --pod ignition-stack \
  -e ACCEPT_IGNITION_EULA=Y \
  -e GATEWAY_ADMIN_PASSWORD=ignition \
  -v ignition-data:/usr/local/bin/ignition/data \
  docker.io/inductiveautomation/ignition:8.3
```

### 5. Verify the installation

- Gateway: http://localhost:8088 (login: `admin` / `ignition`)
- In the Gateway, go to **Config → Databases → Connections**, select the **MariaDB** driver, and use this Connect URL:
  ```
  jdbc:mariadb://localhost:3306/ignition_learning
  ```
  (`localhost` works because the containers are in the same pod and share a network namespace.)

### 6. Run Designer through Toolbox (native desktop window)

Toolbox is included with Fedora. It isolates installed packages from the host while allowing graphical applications to use the desktop's Wayland/X11 display (via XWayland).

**Step 1 — Download the launcher:**
In the Gateway (http://localhost:8088), go to **Downloads → Designer Launcher → Linux** and download `designerlauncher.tar.gz`.

**Step 2 — Create the toolbox** (run this on the host, not inside another toolbox):
```bash
toolbox create ignition-designer
toolbox enter ignition-designer
```
Do not pass `--image`: without it Toolbox picks the image that matches your host release (`fedora-toolbox:45`). The old guide used `fedora-toolbox:41`, which is end-of-life and, on a Fedora 45 host, gave a mismatch with the newer system libraries.

Check with `toolbox list` that the container's image is `fedora-toolbox:45`.

**Step 3 — Install GUI dependencies and extract the archive inside the toolbox.** The home directory is shared with the host, so the downloaded archive is available inside the toolbox:
```bash
sudo dnf install -y gtk3 libXtst libXxf86vm

mkdir -p ~/ignition-designer && cd ~/ignition-designer
tar xzf ~/path/designerlauncher.tar.gz   # Replace this with the archive's actual path
cd designerlauncher
ls                                       # the directory with designerlauncher.sh is the one you need
```

Why these three packages (all required; without them Designer Launcher does not start):

| Package | What happens without it |
|---|---|
| `gtk3` | JavaFX fails with `UnsupportedOperationException: Internal Error` (in `GtkApplication`) |
| `libXtst` | `libglassgtk3.so` cannot load (missing `libXtst.so.6`) |
| `libXxf86vm` | `libprism_es2.so` cannot load (missing `libXxf86vm.so.1`) |

The launcher exits silently or with `Internal Error` and **no window appears** — this is the symptom of missing packages. The log file `designerlauncher.sh.log` is not created in this case; to see the real error run the bundled Java directly:
```bash
./runtime/bin/java -jar app/launcher-designer-linux.jar
```

**Step 4 — Launch Designer:**
```bash
env -u WAYLAND_DISPLAY mesa_glthread=false _JAVA_AWT_WM_NONREPARENTING=1 \
  JAVA_TOOL_OPTIONS="-Dawt.robot.screenshotMethod=x11 -Dsun.java2d.xrender=false" \
  ./designerlauncher.sh
```

What each part does:

| Setting | Purpose |
|---|---|
| `env -u WAYLAND_DISPLAY` | Makes Java treat the session as plain X11 (via XWayland) instead of going through the Wayland screen-sharing portal |
| `mesa_glthread=false` | Prevents a crash of the embedded Chromium (JxBrowser) GPU process: Mesa's `glthread` calls `pthread_setaffinity_np`, which the Chromium sandbox kills with `SIGSYS` |
| `-Dawt.robot.screenshotMethod=x11` | Documented workaround for Designer's popup menus triggering a screen-sharing request on KDE Wayland |
| `-Dsun.java2d.xrender=false` | Disables the Java2D XRender pipeline (common fix for windows that stay blank under XWayland) |
| `_JAVA_AWT_WM_NONREPARENTING=1` | Standard fix for blank Java windows under some window managers |

This **combination** is the configuration confirmed to work on KDE: with `mesa_glthread=false` and `-Dawt.robot.screenshotMethod=x11` but **without** the last two items, Designer opened with a white window. It has not been checked which of `-Dsun.java2d.xrender=false` and `_JAVA_AWT_WM_NONREPARENTING=1` is the one that actually fixes the white window, so keep all of them.

In Designer Launcher, add the Gateway at `localhost:8088` (the toolbox shares the host's network namespace), then sign in with `admin` / `ignition`.

---

## 🛠️ TROUBLESHOOTING

### Launcher does not start / `Internal Error`

Run the bundled Java directly to see the error:
```bash
cd ~/ignition-designer/designerlauncher/designerlauncher
./runtime/bin/java -jar app/launcher-designer-linux.jar
```
Check for missing libraries:
```bash
ldd ~/.openjfx/cache/*/libglassgtk3.so | grep "not found"
ldd ~/.openjfx/cache/*/libprism_es2.so  | grep "not found"
```
(a warning about missing execute permission can be ignored). Each `not found` line names a missing library; find the package with `dnf provides '*/libname.so.N'`.

### Designer window is white

1. Check whether the embedded browser crashed. On the **host** (outside the toolbox):
   ```bash
   coredumpctl list | tail -5
   ```
   A fresh `chromium` crash (`SIGSEGV`, `CrashSIGSYS_Handler`, `pthread_setaffinity_np` in the trace) means `mesa_glthread=false` is missing from the launch command.
2. If there are no crashes and the window is still blank, make sure all five settings from Step 4 are present, especially `-Dsun.java2d.xrender=false` and `_JAVA_AWT_WM_NONREPARENTING=1`.
3. Further options (not needed in the tested setup, try only if 2 did not help): add `-Dsun.java2d.opengl=false -Dsun.java2d.pmoffscreen=false` to `JAVA_TOOL_OPTIONS`, or add `LIBGL_ALWAYS_SOFTWARE=1` to the `env` command (software rendering; slower).

### Dependencies not verified — add only if something fails

These were installed during testing or exist in the default toolbox image, so it is **not known** whether they are required on a clean container. If Designer Launcher or Designer misbehaves, install them:

| Package | Add if you see |
|---|---|
| `alsa-lib` | Errors about `libasound.so` (audio / launcher start) |
| `fontconfig`, `dejavu-sans-fonts` | Missing or garbled text, empty font lists, font-related exceptions |
| `mesa-libGL`, `libglvnd-glx`, `libglvnd-egl`, `mesa-dri-drivers` | OpenGL errors, JavaFX/Chromium rendering failures (normally already in the toolbox image) |
| `libXrender`, `libXi`, `libXcursor`, `libXrandr` | `libX*.so ... not found` for the AWT/JavaFX libraries (normally already in the image) |
| `xdpyinfo` | Only a diagnostic tool: `xdpyinfo \| head -5` checks that the toolbox can reach XWayland. Not needed for Designer itself |

All of the following at once:
```bash
sudo dnf install -y alsa-lib fontconfig dejavu-sans-fonts mesa-libGL \
  libglvnd-glx libglvnd-egl mesa-dri-drivers libXrender libXi libXcursor libXrandr xdpyinfo
```

> Note: `xorg-x11-utils` no longer exists in Fedora 45. If it is in a `dnf install` command, dnf cancels the **whole** transaction and installs nothing.

### `toolbox rm` says "container is running"

You are probably still inside that toolbox. Run `exit` to return to the host, then:
```bash
toolbox rm --force ignition-designer
```
(`toolbox stop` does not exist; `podman stop ignition-designer` also works.)

### Known issues

_To be documented._

---

## 🗑️ COMPLETE REMOVAL

```bash
# Remove the Gateway and MariaDB containers; the volumes (data) are retained
podman pod rm ignition-stack --force

# Remove Gateway and MariaDB data (full reset)
podman volume rm ignition-data mariadb-data

# Remove the Designer toolbox (run on the host, not inside the toolbox)
toolbox rm --force ignition-designer
```
