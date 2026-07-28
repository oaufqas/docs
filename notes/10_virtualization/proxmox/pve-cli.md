**`pve-cli`** — это общее неофициальное название мощного набора встроенных утилит командной строки (CLI) для управления гипервизором Proxmox VE.

В то время как веб-панель Proxmox работает на порту `8006`, под капотом гипервизора крутится обычный Linux (Debian). Все действия, которые вы кликаете мышкой в браузере, Proxmox на самом деле выполняет через эти самые консольные команды.


В Proxmox система разбита на специализированные утилиты:

1. **`qm` (Qemu Manager)** — управление полноценными виртуальными машинами (KVM).
2. **`pct` (Proxmox Container Toolkit)** — управление LXC-контейнерами.
3. **`pvesm` (Proxmox Storage Manager)** — управление дисками, LVM и хранилищами.
4. **`pveum` (Proxmox User Manager)** — управление пользователями, правами и API-токенами.
5. **`pvecm` (Proxmox Cluster Manager)** — сборка серверов в единый кластер.

---

> Qemu Manager `qm`:

- `qm list` — вывести список всех виртуальных машин (покажет их ID, статус, RAM).
- `qm start 101` — запустить виртуалку .
- `qm stop 101` — жестко выключить виртуалку (выдернуть «виртуальный провод»).
- `qm shutdown 101` — мягко послать сигнал выключения через ACPI/QEMU Guest Agent.
- `qm reboot 101` — отправить машину в перезагрузку.
- `qm config 101` — прочитать весь текстовый конфигурационный файл виртуалки (вы увидите все диски, сети и процессоры).
- `qm status 101` — проверить, запущена ли машина прямо сейчас.
- `qm clone 100 101 --full` — мгновенно создать полный клон виртуалки из шаблона (VMID 100) с новым ID 101.
- `qm template 101` — превратить готовую виртуалку в неизменяемый эталонный шаблон.
- `qm destroy 101` — полностью удалить виртуалку и стереть её диски с SSD.

```bash
qm create 110 --name "ubuntu-template" --memory 2048 --cores 1 --net0 virtio,bridge=vmbr0 # Создание оболочки виртуальной машины

qm importdisk 110 /var/lib/vz/template/iso/noble-server-cloudimg-amd64.img local-lvm # Создание диска и путь к образу

qm set 110 --scsihw virtio-scsi-single --scsi0 local-lvm:vm-110-disk-0,discard=on # Прикрепление диска к машине

qm set 110 --ide2 local-lvm:cloudinit # Прикрепление cloud-init диска

qm set 110 --boot c --bootdisk scsi0 # Указание загрузочного диска

qm set 110 --agent enabled=1 # Включение qm агента, работающего в виртуалке

qm resize 110 scsi0 +7,5G # Увеличение размера диска
```

> Proxmox Container Toolkit `pct`:

- `pct list` — список всех запущенных контейнеров.
- `pct start 111` — запустить контейнер.
- `pct enter 111` — Провалиться в консоль контейнера напрямую из хоста Proxmox без ввода паролей и SSH.
- `pct stop 111` — остановить контейнер.


> Proxmox Storage Manager `pvesm`:

- `pvesm status` — показать статус всех дисков (LVM-Thin, папок), сколько места свободно на SSD.
- `pvesm alloc local-lvm 101 vm-101-disk-0 20G` — руками создать сырой пустой диск размером 20 ГБ внутри тонкого пула `local-lvm` для виртуалки 101.
- `pvesm path local:iso/ubuntu-22.04-live-server-amd64.iso` — показать точный абсолютный путь в Linux-системе, где физически лежит скачанный ISO-образ.

> Proxmox User Manager `pveum`:

- `pveum user list` — список всех пользователей в системе.
- `pveum user add terraform-user@pve` — создать нового пользователя.
- `pveum user token add terraform-user@pve tf-token --privsep 0` — выпустить API-токен автоматизации.
- `pveum aclmodify / -user terraform-user@pve -role PVEVMAdmin` — выдать права на управление виртуалками.

> Системные команды самого Proxmox

- `pveversion -v` — показать детальные версии всех ядер (ядро Linux, версия pve-manager, версия QEMU). Критично проверять при обновлениях.
- `systemctl restart pvedaemon pveproxy` — перезапустить движок и веб-интерфейс Proxmox (помогает, если зависла веб-панель, при этом сами виртуалки не выключаются и продолжают работать!).

---