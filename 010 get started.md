# Встановлення Ignition на Fedora Server (VMware Workstation 17 Pro)

## 1. Підготовка VM (VMware Workstation 17 Pro)

Налаштування вручну через GUI:
- vCPU: 4
- RAM: 8 GB
- Диск: 40-60 GB, "Allocate all disk space now" (Thick Provision Eagerly Zeroed)
- SCSI-контролер: LSI Logic SAS
- Мережевий адаптер: VMXNET3
- Встановлена ОС: Fedora Server

## 2. Встановлення VMware Tools на Fedora

```bash
sudo dnf install open-vm-tools -y
sudo systemctl enable --now vmtoolsd
```

## 3. Перевірка/увімкнення SSH (для доступу з Windows)

```bash
sudo systemctl status sshd
sudo systemctl enable --now sshd
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

## 4. Завантаження інсталятора Ignition

На Windows:
1. Відкрити https://inductiveautomation.com/downloads/ignition/
2. Вибрати Linux 64-bit
3. Завантажити `.run`-файл

Перенести файл на сервер через scp (з PowerShell на Windows):

```powershell
scp "C:\Users\<user>\Downloads\ignition-8.3.8-linux-64-installer.run" username@<server-ip>:/home/username/
```

## 5. Зробити інсталятор виконуваним

```bash
chmod +x ignition-8.3.8-linux-64-installer.run
```

## 6. Створити системного користувача ignition

```bash
sudo useradd -r -m -d /var/lib/ignition -s /sbin/nologin ignition
```

## 7. Запустити інсталяцію (unattended-режим)

```bash
sudo ./ignition-8.3.8-linux-64-installer.run -- "user=ignition" "autoStart=true" "unattended=minimal"
```

## 8. Перевірка статусу сервісу

```bash
sudo /usr/local/bin/ignition/ignition.sh status
systemctl status Ignition-Gateway
```

> Назва сервісу — саме `Ignition-Gateway`, не `ignition`.

## 9. Відкрити порти у firewalld

```bash
sudo firewall-cmd --permanent --add-port=8088/tcp --add-port=8043/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

## 10. Доступ до Gateway з браузера (Windows)

Дізнатись IP сервера:

```bash
ip addr show
```

Відкрити в браузері:

```
http://<IP-сервера>:8088
```

## 11. Gateway Setup Wizard

У браузері вибрати редакцію **Ignition** (не Maker/Edge) і пройти майстер налаштування:
- Прийняти EULA
- Створити адміністративного користувача
- Вказати назву Gateway