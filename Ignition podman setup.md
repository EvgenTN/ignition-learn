# Ignition + MariaDB у Podman (Fedora, learning)

---

## 🟢 ЩОДЕННЕ ВИКОРИСТАННЯ (коротка шпаргалка)

Це все, що треба пам'ятати, коли все вже встановлено (розділи нижче — "ВСТАНОВЛЕННЯ").

**Підняти Gateway + MariaDB:**
```bash
podman pod start ignition-stack
```
Перевірити: http://localhost:8088 (логін `admin` / `ignition`)

**Зупинити Gateway + MariaDB** (дані нікуди не діваються):
```bash
podman pod stop ignition-stack
```

**Перевірити, що зараз запущено:**
```bash
podman pod ps
podman ps --pod
```

**Відкрити Designer:**
```bash
toolbox enter ignition-designer
```
```bash
cd ~/ignition-designer/designerlauncher
./designerlauncher.sh
exit
```
(запускаємо у передньому плані, одразу після — `exit`; саме ця послідовність стабільно відкриває вікно на потрібному моніторі)

**Зайти в MariaDB консоллю (для налагодження):**
```bash
podman exec -it mariadb mariadb -u ignition -pignition ignition_learning
```

---

## ⚙️ ВСТАНОВЛЕННЯ (один раз)

### 1. Volume-и для Gateway і MariaDB

Щоб дані (проєкти, теги, таблиці) переживали видалення/пересворення контейнера.

```bash
podman volume create ignition-data
podman volume create mariadb-data
```

### 2. Pod (щоб керувати Gateway і MariaDB як єдиним цілим)

Всі порти публікуються на рівні pod-а — контейнери всередині бачать одна одну як `localhost`.

```bash
podman pod create --name ignition-stack \
  -p 8088:8088 \
  -p 8043:8043 \
  -p 3306:3306
```

### 3. MariaDB

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

### 4. Ignition Gateway (8.3)

```bash
podman run -d --name ignition-gw \
  --pod ignition-stack \
  -e ACCEPT_IGNITION_EULA=Y \
  -e GATEWAY_ADMIN_PASSWORD=ignition \
  -v ignition-data:/usr/local/bin/ignition/data \
  docker.io/inductiveautomation/ignition:8.3
```

### 5. Перевірка встановлення

- Gateway: http://localhost:8088 (логін: `admin` / `ignition`)
- У Gateway → Config → Databases → Connections → драйвер **MariaDB**, Connect URL:
  ```
  jdbc:mariadb://localhost:3306/ignition_learning
  ```
  (`localhost`, бо контейнери в одному pod-і й ділять мережевий namespace)

### 6. Designer через Toolbox (нативне вікно на робочому столі)

Toolbox — вбудований у Fedora інструмент, ізолює встановлені пакети від хоста, але прокидає Wayland/X11 напряму, тож застосунок відкривається як звичайне вікно на робочому столі.

**Крок 1** — завантаж лаунчер:
На Gateway (http://localhost:8088) → **Downloads → Designer Launcher → Linux** → завантаж `designerlauncher.tar.gz`.

**Крок 2** — створи контейнер:
```bash
toolbox create --image registry.fedoraproject.org/fedora-toolbox:41 ignition-designer
toolbox enter ignition-designer
```

**Крок 3** — всередині toolbox постав GUI-залежності і розпакуй архів (домашня тека спільна з хостом, тож файл вже доступний за шляхом, куди його зберіг браузер):
```bash
# без цього Designer Launcher падає з "Internal Error"
sudo dnf install -y gtk3 mesa-libGL libXtst libXrender alsa-lib

mkdir -p ~/ignition-designer && cd ~/ignition-designer
tar xzf ~/path/designerlauncher.tar.gz   # заміни на реальний шлях
cd designerlauncher
```

**Крок 4** — перший запуск і закріплення налаштування:
```bash
GDK_BACKEND=x11 ./designerlauncher.sh
```
`GDK_BACKEND=x11` обов'язковий — Fedora Workstation за замовчуванням Wayland, а Glass/JavaFX-бекенд лаунчера з ним падає. Щоб не вводити щоразу:
```bash
echo 'export GDK_BACKEND=x11' >> ~/.bashrc
```

У вікні Designer Launcher додай Gateway адресою `localhost:8088` (toolbox ділить мережевий namespace з хостом), логін `admin` / `ignition`.

---

## 🗑️ ПОВНЕ ВИДАЛЕННЯ

```bash
# Gateway + MariaDB (контейнери; volume-и, тобто дані, лишаються)
podman pod rm ignition-stack --force

# дані Gateway + MariaDB (повний скид)
podman volume rm ignition-data mariadb-data

# Designer
toolbox rm ignition-designer
```
