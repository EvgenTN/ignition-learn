# Ignition + MariaDB with Podman (Fedora, learning)

---

## 🟢 DAILY USE (quick reference)

This is all you need to remember once everything is installed (the sections below cover installation).

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
cd ~/ignition-designer/designerlauncher
env -u WAYLAND_DISPLAY GDK_BACKEND=x11 ./designerlauncher.sh
```
Run the launcher in the foreground. After closing Designer, run `exit` to leave the toolbox. Unsetting `WAYLAND_DISPLAY` and selecting the X11 backend avoids the screen-sharing dialog that appeared instead of the expected context menu in this setup.

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

Toolbox is included with Fedora. It isolates installed packages from the host while allowing graphical applications to use the desktop's Wayland/X11 display.

**Step 1 — Download the launcher:**
In the Gateway (http://localhost:8088), go to **Downloads → Designer Launcher → Linux** and download `designerlauncher.tar.gz`.

**Step 2 — Create the toolbox:**
```bash
toolbox create --image registry.fedoraproject.org/fedora-toolbox:41 ignition-designer
toolbox enter ignition-designer
```

**Step 3 — Install GUI dependencies and extract the archive inside the toolbox.** The home directory is shared with the host, so the downloaded archive is available inside the toolbox:
```bash
# Required to avoid an "Internal Error" when starting Designer Launcher
sudo dnf install -y gtk3 mesa-libGL libXtst libXrender alsa-lib

mkdir -p ~/ignition-designer && cd ~/ignition-designer
tar xzf ~/path/designerlauncher.tar.gz   # Replace this with the archive's actual path
cd designerlauncher
```

**Step 4 — Launch Designer:**
```bash
env -u WAYLAND_DISPLAY GDK_BACKEND=x11 ./designerlauncher.sh
```
The `env` command removes `WAYLAND_DISPLAY` for this launch and sets `GDK_BACKEND=x11`. This is the launch configuration confirmed to work in this setup. Run the launcher from `~/ignition-designer/designerlauncher` inside the `ignition-designer` toolbox.

In Designer Launcher, add the Gateway at `localhost:8088` (the toolbox shares the host's network namespace), then sign in with `admin` / `ignition`.

---

## 🗑️ COMPLETE REMOVAL

```bash
# Remove the Gateway and MariaDB containers; the volumes (data) are retained
podman pod rm ignition-stack --force

# Remove Gateway and MariaDB data (full reset)
podman volume rm ignition-data mariadb-data

# Remove the Designer toolbox
toolbox rm ignition-designer
```
