# netrix-switch-build — Roadmap

План разработки по фазам. Формат: `- [ ]` не начато, `- [~]` в работе, `- [x]` готово.

---

## Фаза 1 — Docker build environment ⏳

### 1.1 Каркас проекта

- [x] `netrix-switch-build/` директория
- [x] `AGENTS.md` — описание проекта
- [x] `ROADMAP.md` — этот файл
- [ ] `Makefile` — базовые targets (docker-build, docker-shell, docker-clean)
- [ ] `Dockerfile` — Debian trixie с build-зависимостями
- [ ] `scripts/common.sh` — общие функции (логирование, пути)

### 1.2 Docker образ

- [ ] `netrix:bld` — базовый build-контейнер
- [ ] Проверка: `make docker-build && make docker-shell`
- [ ] Проверка: внутри gcc, cmake, protobuf-c, libyang, все .h на месте

### 1.3 Sources

- [ ] `sources/linux` → symlink в references
- [ ] `sources/sdk-6.5.24` → symlink в borandcom/sdk
- [ ] `sources/saibcm-modules` → symlink в borandcom/sonic
- [ ] `sources/danos` → symlink в references
- [ ] `sources/netrix-confd` → symlink в references
- [ ] `sources/libsaibcm.deb` → Broadcom SAI binary (на случай теста)

---

## Фаза 2 — Linux kernel ⏳

### 2.1 Kernel source

- [ ] `scripts/01-fetch-kernel.sh` — clone linux-6.1.x или sonic-linux-kernel
- [ ] `patches/kernel/series` — quilt-style series
- [ ] `patches/kernel/00XX-*.patch` — патчи под BES-53248A1-G1 (если нужны)

### 2.2 saibcm-modules

- [ ] `scripts/02-build-saibcm-modules.sh` — собрать kernel modules
- [ ] `linux-kernel-bde.ko`, `linux-user-bde.ko`, `ngbde.ko`, `ngknet.ko`
- [ ] DKMS-пакет или обычный .deb

### 2.3 Kernel .deb

- [ ] `scripts/03-build-kernel.sh` — cross/native сборка
- [ ] `linux-image-*.deb`, `linux-headers-*.deb`
- [ ] `linux-firmware-*.deb` (для сетевых чипов)

---

## Фаза 3 — libopennsa (Broadcom SDK) ⏳

### 3.1 Подготовка SDK

- [ ] `scripts/04-prepare-sdk.sh` — apply наши патчи к SDK 6.5.24
- [ ] `patches/sdk/00XX-*.patch` — патчи что мы уже сделали (gcc 15 fixes)

### 3.2 Сборка

- [ ] `scripts/05-build-opennsa.sh` — собрать user-space библиотеку
- [ ] `libopennsa.so` (589 MB), `libopennsa.a` (1.17 GB)
- [ ] Заголовки в `/usr/include/opennsa/`

### 3.3 Упаковка

- [ ] `debian/control` — `libopennsa1`, `libopennsa-dev`
- [ ] Provides: `libopennsl1`, `libopennsl-dev` (для совместимости)
- [ ] `.deb` файлы в `output/debs/`

---

## Фаза 4 — libfal-opennsl ⏳

### 4.1 Подготовка

- [ ] `scripts/06-prepare-fal.sh` — взять `libfal-opennsl` из DANOS
- [ ] Sed-переименование `NSL_*` → `NSA_*`, `opennsl_*` → `opennsa_*`
- [ ] Проверка: все вызовы резолвятся в libopennsa

### 4.2 Сборка

- [ ] `scripts/07-build-fal.sh` — собрать FAL-плагин
- [ ] `libfal-opennsl.so.1` (требует libopennsa + libdpdk)

### 4.3 Упаковка

- [ ] `libfal-opennsl1` (Depends: libopennsa1)
- [ ] `libfal-opennsl-dev` (Depends: libopennsa-dev, libvyattafal-dev)

---

## Фаза 5 — netrix-confd ⏳

### 5.1 Сборка из references

- [ ] `scripts/08-build-netrix-confd.sh` — cmake build из sources/netrix-confd
- [ ] `nconfd`, `ncli`, `yang2nsb`, `show_platform` бинарники
- [ ] `schema.bin` скомпилирован из YANG

### 5.2 Упаковка

- [ ] `netrix-confd` (Depends: libopennsa1, libfal-opennsl1)
- [ ] `/usr/sbin/nconfd`, `/usr/bin/ncli`
- [ ] `/usr/share/netrix/schema.bin`
- [ ] `/usr/libexec/netrix/scripts/conf/*`, `op/*`, `completion/*`
- [ ] systemd unit: `nconfd.service`

---

## Фаза 6 — Platform driver (BES-53248A1-G1) ⏳

### 6.1 Специфика железа

- [ ] `platform/bes-53248a1/` — специфика нашего свича
- [ ] port-config: 48×25G SFP28 + 8×100G QSFP28
- [ ] LED processor firmware (`port_locator.c`)
- [ ] Fan/PSU/SFP control (C/Perl скрипты)
- [ ] EEPROM данные

### 6.2 Конфиги

- [ ] `etc/bcm/flex/bcm56870_a0_premium_issu/b870.5.3.3/` (наш flex pkg)
- [ ] `etc/bcm/xgs/td3_ix8_10Gx48_40Gx8.config.bcm` или свой port layout
- [ ] `platform.conf` для libfal-opennsl

### 6.3 Упаковка

- [ ] `netrix-platform-bes53248a1` .deb
- [ ] Post-install: создание `/etc/bcm/`, копирование flex pkg

---

## Фаза 7 — Base rootfs ⏳

### 7.1 Debootstrap

- [ ] `scripts/09-create-base-rootfs.sh` — debootstrap trixie
- [ ] `output/rootfs/` — базовая Debian система

### 7.2 System overlay

- [ ] `configs/system/etc/` — конфиги (systemd, network, ssh, fstab)
- [ ] `configs/system/root/` — root home

### 7.3 Packages

- [ ] `scripts/10-install-packages.sh` — ставит всё нужное
- [ ] kernel + saibcm-modules + libopennsa + libfal-opennsl + netrix-confd + платформенные
- [ ] Системные: systemd, ssh, network-manager (или none), bash-completion

---

## Фаза 8 — ONIE installer ⏳

### 8.1 ONIE совместимость

- [ ] `scripts/11-create-onie-image.sh` — собрать ONIE installer
- [ ] GRUB с ONIE режимом
- [ ] Recovery/Install/Uninstall modes

### 8.2 Тестовый образ

- [ ] `output/images/netrix-bes53248a1-v0.1.0.bin`
- [ ] Rescue shell
- [ ] Auto-install через DHCP/TFTP

---

## Фаза 9 — Первый запуск на железе ⏳

### 9.1 Smoke test

- [ ] Чип detect'ится как BCM56970
- [ ] bcm_init успешно (flex 5.3.3)
- [ ] Порты поднимаются
- [ ] FAL plugin грузится
- [ ] netrix-confd стартует

### 9.2 Functional test

- [ ] `ncli set interfaces ethernet eth0 address 192.168.1.1/24`
- [ ] `ncli commit`
- [ ] `ncli show interfaces`
- [ ] `ping` через свич
- [ ] L2 forwarding работает

---

## Фаза 10 — Расширение ⏳

### 10.1 Дополнительные YANG

- [ ] Порт `vyatta-interfaces-dataplane-v1.yang` → `netrix-interfaces-dataplane.yang`
- [ ] Порт `vyatta-security-storm-control-v1.yang` → `netrix-storm-control.yang`
- [ ] Порт `vyatta-vrf-*` → `netrix-vrf.yang`
- [ ] Добавить конфиги в schema.bin

### 10.2 Скрипты

- [ ] `scripts/conf/interfaces-ethernet` — apply через FAL
- [ ] `scripts/conf/interfaces-switch` — L2 forwarding
- [ ] `scripts/conf/storm-control` — rate limiting
- [ ] `scripts/op/show-interfaces` — show status

### 10.3 Безопасность

- [ ] NACM (RFC 8341) — реализация в netrix-confd
- [ ] Audit log (append-only, signed)
- [ ] rlimit для commit-скриптов

---

## Фаза 11 — Production hardening ⏳

### 11.1 HA (опционально)

- [ ] Два свича в стеке
- [ ] State sync
- [ ] Failover

### 11.2 NETCONF

- [ ] NETCONF server поверх nconfd (для SDN)
- [ ] gNMI/telemetry (опционально)

### 11.3 Мониторинг

- [ ] SNMP интеграция
- [ ] Prometheus exporter
- [ ] Hardware telemetry (sensors, fans, PSU)

---

## Легенда приоритетов

- 🔴 критично для базового запуска
- 🟡 важно для production
- 🟢 nice to have

| Фаза | Приоритет | Статус |
| --- | --- | --- |
| 1 — Docker | 🔴 | ⏳ |
| 2 — Kernel + saibcm | 🔴 | ⏳ |
| 3 — libopennsa | 🔴 | ⏳ |
| 4 — libfal-opennsl | 🔴 | ⏳ |
| 5 — netrix-confd | 🔴 | ⏳ |
| 6 — Platform | 🔴 | ⏳ |
| 7 — Rootfs | 🔴 | ⏳ |
| 8 — ONIE | 🟡 | ⏳ |
| 9 — Первый запуск | 🔴 | ⏳ |
| 10 — Расширение | 🟡 | ⏳ |
| 11 — Production | 🟢 | ⏳ |
