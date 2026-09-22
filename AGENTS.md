# netrix (project root)

Корневая директория проекта netrix — сетевая ОС на базе Debian trixie.
Целевое устройство: **Lenovo BES-53248A1-G1** (BCM56970 Trident3-X8, 48×25G + 8×100G).

## Архитектура

- **Хост сборки**: x86_64 (native, без cross-compile)
- **Базовый образ**: Debian trixie
- **Подход**: Docker-контейнер с build-зависимостями → Debian rootfs через debootstrap → ONIE installer
- **Стек софта**:
  - Linux kernel 6.1+ (x86_64) с модулями saibcm (linux-kernel-bde, ngbde, ngknet)
  - libopennsa.so (собрана из Broadcom SDK 6.5.24 source, OpenNSA license)
  - libfal-opennsl.so (DANOS, с переименованием NSL_* → NSA_*)
  - netrix-confd (C-демон, protobuf-c, flat schema.bin)
  - DPDK + vPlane (dataplane forwarding)

## Структура корня

```text
/srv/git/projects/netrix/
├── AGENTS.md              # этот файл
├── ROADMAP.md             # план по фазам
├── Dockerfile             # netrix:bld — build-окружение
├── Makefile               # оркестрация сборки
├── scripts/               # 01-10 fetch/build/create
├── configs/               # apt/, system/{etc,root}/ overlays
├── sources/               # symlinks → borandcom/, references/...
├── patches/               # kernel/, sdk/
├── platform/              # специфика свича (BES-53248A1-G1)
├── output/                # kernel/, debs/, rootfs/, images/
├── borandcom/             # SDK 6.5.24 + SONiC + saibcm-modules
├── danos/                 # (источник для libfal-opennsl)
├── references/            # clixon, ksdr-fw, netrix-confd, sonic, dan
├── libsaibcm_*.deb        # Broadcom SAI binary (из Microsoft CDN)
└── .build-logs/           # логи сборок
```

## Сборка (этап 1 — Docker)

```bash
cd /srv/git/projects/netrix

make docker-build         # собрать netrix:bld
make docker-shell         # зайти в контейнер
make docker-clean         # удалить образ
```

## Сборка (этап 2+ — добавлено позже)

```bash
make build-kernel         # Linux kernel + .deb
make build-opennsa        # Broadcom SDK 6.5.24 → libopennsa.so + .deb
make build-fal            # libfal-opennsl.so + .deb (NSL→NSA)
make build-netrix-confd   # наш C-демон + .deb
make build-platform       # BES-53248A1-G1 platform driver + .deb
make create-base-rootfs   # debootstrap Debian trixie rootfs
make install-packages     # поставить всё в rootfs
make create-onie-image    # финальный ONIE installer
```

## Зависимости

Все build-зависимости ставятся внутри Docker. Снаружи нужен только:

- Docker Engine 24+
- Docker Buildx

## Sources (symlinks)

`sources/` содержит симлинки на исходники:

```text
sources/linux          → /srv/git/projects/netrix/borandcom/linux
sources/sdk-6.5.24     → /srv/git/projects/netrix/borandcom/sdk/sdk-6.5.24
sources/saibcm-modules → /srv/git/projects/netrix/borandcom/sonic/platform/broadcom/saibcm-modules
sources/danos          → /srv/git/projects/netrix/references/danos
sources/netrix-confd   → /srv/git/projects/netrix/references/netrix-confd
```

## Лицензия

См. `LICENSE` (TBD: GPL v3 или proprietary).

## Code style

- Shell: bash с `set -e`, цветной вывод через ANSI
- Makefile: `.PHONY` для всех targets, цветной вывод
- C: см. `references/netrix-confd/.clang-format` (LLVM, 4-space indent)
