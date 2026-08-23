# Ignition + MariaDB у Podman (Fedora, learning stack "Receiving")

Прості паролі — лише для локального навчального стенду, не для проду.

## 1. Volume-и (щоб дані переживали rm/пересворення контейнера)

```bash
podman volume create ignition-data
podman volume create mariadb-data
```

## 2. Pod (щоб керувати обома контейнерами разом)

Всі порти публікуються на рівні pod-а — контейнери всередині бачать одна одну як `localhost`.

```bash
podman pod create --name ignition-stack \
  -p 8088:8088 \
  -p 8043:8043 \
  -p 3306:3306
```

## 3. MariaDB

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

## 4. Ignition Gateway (8.3)

```bash
podman run -d --name ignition-gw \
  --pod ignition-stack \
  -e ACCEPT_IGNITION_EULA=Y \
  -e GATEWAY_ADMIN_PASSWORD=ignition \
  -v ignition-data:/usr/local/bin/ignition/data \
  docker.io/inductiveautomation/ignition:8.3
```

## 5. Перевірка

- Gateway: http://localhost:8088 (логін: `admin` / `ignition`)
- MariaDB (клієнт вже є всередині контейнера, нічого на хості ставити не треба):
  ```bash
  podman exec -it mariadb mariadb -u ignition -pignition ignition_learning
  ```
  (у MariaDB 11+ клієнтський бінарник називається `mariadb`, а не `mysql`)
- У Gateway → Config → Databases → Connections → драйвер **MariaDB**, Connect URL:
  ```
  jdbc:mariadb://localhost:3306/ignition_learning
  ```
  (`localhost`, бо контейнери в одному pod-і й ділять мережевий namespace)

## 6. Керування всім стеком одразу

```bash
# зупинити обидва контейнери
podman pod stop ignition-stack

# запустити знову (дані збережені у volume-ах)
podman pod start ignition-stack

# статус
podman pod ps
podman ps --pod

# повністю прибрати контейнери (volume-и лишаються)
podman pod rm ignition-stack --force

# прибрати і дані (повний скид)
podman volume rm ignition-data mariadb-data
```

Автостарт при завантаженні хоста навмисно не налаштований — стек піднімається вручну командою `podman pod start ignition-stack`, коли потрібно попрацювати.