# NixOS Configurations

Репозиторий для хранения конфигураций нескольких NixOS-машин.

В этом примере используются две машины:

- `desktop` — основная рабочая машина;
- `server` — сервер.

Конфигурации обеих машин хранятся в одном Git-репозитории, но находятся в разных каталогах.

Общие настройки выносятся в отдельные модули, чтобы не дублировать код.

---

# Содержание

- [Что получится в итоге](#что-получится-в-итоге)
- [Требования](#требования)
- [Установка Git](#установка-git)
- [Создание репозитория](#создание-репозитория)
- [Структура репозитория](#структура-репозитория)
- [Создание файлов](#создание-файлов)
- [Настройка Flakes](#настройка-flakes)
- [Создание конфигурации основной машины](#создание-конфигурации-основной-машины)
- [Создание конфигурации сервера](#создание-конфигурации-сервера)
- [Перенос hardware-конфигурации](#перенос-hardware-конфигурации)
- [Общие модули](#общие-модули)
- [Применение конфигурации](#применение-конфигурации)
- [Проверка конфигурации](#проверка-конфигурации)
- [Работа с Git](#работа-с-git)
- [Синхронизация машин](#синхронизация-машин)
- [Обновление NixOS](#обновление-nixos)
- [Добавление новых пакетов](#добавление-новых-пакетов)
- [Добавление новых сервисов](#добавление-новых-сервисов)
- [Настройка SSH-сервера](#настройка-ssh-сервера)
- [Настройка Docker](#настройка-docker)
- [Настройка Firewall](#настройка-firewall)
- [Работа с секретами](#работа-с-секретами)
- [Откат конфигурации](#откат-конфигурации)
- [Добавление новой машины](#добавление-новой-машины)
- [Рекомендуемый рабочий процесс](#рекомендуемый-рабочий-процесс)

---

# Что получится в итоге

Итоговая структура репозитория будет выглядеть так:

```text
nixos-config/
├── flake.nix
├── flake.lock
├── README.md
├── .gitignore
├── switch.sh
│
├── hosts/
│   ├── desktop/
│   │   ├── default.nix
│   │   └── hardware-configuration.nix
│   │
│   └── server/
│       ├── default.nix
│       └── hardware-configuration.nix
│
├── modules/
│   ├── common.nix
│   ├── desktop.nix
│   ├── server.nix
│   ├── ssh.nix
│   ├── docker.nix
│   └── firewall.nix
│
├── home/
│   ├── desktop.nix
│   └── server.nix
│
└── secrets/
    └── .gitkeep
```

Для основной машины будет использоваться команда:

```bash
sudo nixos-rebuild switch --flake .#desktop
```

Для сервера:

```bash
sudo nixos-rebuild switch --flake .#server
```

---

# Требования

Для работы понадобится:

- установленный NixOS;
- установленный Git;
- доступ к GitHub или другому Git-серверу;
- пользователь с правами `sudo`;
- включённые Flakes;
- одинаковая или совместимая версия NixOS на машинах.

Проверить версию NixOS:

```bash
nixos-version
```

Проверить архитектуру системы:

```bash
uname -m
```

Для большинства компьютеров и серверов результат будет:

```text
x86_64
```

Для ARM-систем результат будет:

```text
aarch64
```

---

# Установка Git

Если Git уже установлен, этот раздел можно пропустить.

Проверить наличие Git:

```bash
git --version
```

Если Git не установлен, временно запустить shell с Git:

```bash
nix-shell -p git
```

После этого команда `git` будет доступна только в текущем shell.

Для постоянной установки Git добавьте его в конфигурацию NixOS:

```nix
environment.systemPackages = with pkgs; [
  git
];
```

После этого примените конфигурацию:

```bash
sudo nixos-rebuild switch
```

---

# Создание репозитория

## Вариант 1. Создание репозитория через GitHub

Создайте новый пустой репозиторий на GitHub.

Например:

```text
nixos-config
```

Рекомендуется не создавать дополнительные файлы при создании репозитория:

- README;
- `.gitignore`;
- LICENSE.

Эти файлы будут созданы локально.

---

## Клонирование репозитория

Перейдите в домашний каталог:

```bash
cd ~
```

Клонируйте репозиторий:

```bash
git clone https://github.com/<пользователь>/<репозиторий>.git nixos-config
```

Пример:

```bash
git clone https://github.com/example/nixos-config.git nixos-config
```

Перейдите в каталог:

```bash
cd ~/nixos-config
```

Проверьте содержимое:

```bash
ls -la
```

---

## Вариант 2. Создание репозитория локально

Если удалённый репозиторий ещё не создан:

```bash
mkdir -p ~/nixos-config
cd ~/nixos-config
```

Инициализируйте Git:

```bash
git init
```

После создания репозитория на GitHub добавьте удалённый адрес:

```bash
git remote add origin https://github.com/<пользователь>/<репозиторий>.git
```

Проверить удалённый репозиторий:

```bash
git remote -v
```

---

# Создание структуры репозитория

Находясь в каталоге репозитория, выполните:

```bash
cd ~/nixos-config
```

Создайте каталоги:

```bash
mkdir -p hosts/desktop
mkdir -p hosts/server
mkdir -p modules
mkdir -p home
mkdir -p secrets
```

Создайте файл `.gitkeep`:

```bash
touch secrets/.gitkeep
```

Создайте основные файлы:

```bash
touch flake.nix
touch README.md
touch .gitignore
touch switch.sh
```

Итоговая структура на этом этапе:

```text
nixos-config/
├── flake.nix
├── README.md
├── .gitignore
├── switch.sh
├── hosts/
│   ├── desktop/
│   └── server/
├── modules/
├── home/
└── secrets/
    └── .gitkeep
```

---

# Включение Flakes

Flakes нужны для управления конфигурациями нескольких машин.

Откройте текущую системную конфигурацию:

```bash
sudo nano /etc/nixos/configuration.nix
```

Добавьте:

```nix
nix.settings.experimental-features = [
  "nix-command"
  "flakes"
];
```

Пример минимальной конфигурации:

```nix
{ config, pkgs, ... }:

{
  nix.settings.experimental-features = [
    "nix-command"
    "flakes"
  ];

  environment.systemPackages = with pkgs; [
    git
    vim
  ];

  system.stateVersion = "25.05";
}
```

Примените изменения:

```bash
sudo nixos-rebuild switch
```

Проверьте, что Flakes работают:

```bash
nix flake --help
```

Если команда выводит справку, Flakes включены.

---

# Создание файла `flake.nix`

Откройте файл:

```bash
nano ~/nixos-config/flake.nix
```

Добавьте следующий код:

```nix
{
  description = "NixOS configurations";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
  };

  outputs = { self, nixpkgs, ... }:
    {
      nixosConfigurations = {
        desktop = nixpkgs.lib.nixosSystem {
          system = "x86_64-linux";

          modules = [
            ./hosts/desktop
          ];
        };

        server = nixpkgs.lib.nixosSystem {
          system = "x86_64-linux";

          modules = [
            ./hosts/server
          ];
        };
      };
    };
}
```

## Важное замечание

В примере используется:

```nix
nixos-unstable
```

Для стабильной системы лучше использовать ветку, соответствующую установленной версии NixOS.

Например:

```nix
nixpkgs.url = "github:NixOS/nixpkgs/nixos-25.05";
```

Или другую актуальную стабильную ветку.

Посмотреть установленную версию:

```bash
nixos-version
```

Если вывод начинается, например, с:

```text
25.05
```

можно использовать:

```nix
nixpkgs.url = "github:NixOS/nixpkgs/nixos-25.05";
```

---

# Архитектура машин

Если обе машины используют Intel или AMD, оставьте:

```nix
system = "x86_64-linux";
```

Если машина использует ARM, укажите:

```nix
system = "aarch64-linux";
```

Например:

```nix
server = nixpkgs.lib.nixosSystem {
  system = "aarch64-linux";

  modules = [
    ./hosts/server
  ];
};
```

Если архитектуры разные, можно указать разные значения:

```nix
nixosConfigurations = {
  desktop = nixpkgs.lib.nixosSystem {
    system = "x86_64-linux";

    modules = [
      ./hosts/desktop
    ];
  };

  server = nixpkgs.lib.nixosSystem {
    system = "aarch64-linux";

    modules = [
      ./hosts/server
    ];
  };
};
```

---

# Создание общей конфигурации

Создайте файл:

```bash
nano ~/nixos-config/modules/common.nix
```

Добавьте:

```nix
{ pkgs, ... }:

{
  nix.settings.experimental-features = [
    "nix-command"
    "flakes"
  ];

  nix.settings.auto-optimise-store = true;

  networking.networkmanager.enable = true;

  environment.systemPackages = with pkgs; [
    vim
    git
    curl
    wget
    htop
    tree
    tmux
  ];

  security.sudo.wheelNeedsPassword = true;

  nix.gc = {
    automatic = true;
    dates = "weekly";
    options = "--delete-older-than 14d";
  };
}
```

Этот модуль будет использоваться на обеих машинах.

В него можно помещать настройки, которые подходят и для основной машины, и для сервера.

---

# Создание конфигурации основной машины

## Создание файла `hosts/desktop/default.nix`

Создайте файл:

```bash
nano ~/nixos-config/hosts/desktop/default.nix
```

Добавьте:

```nix
{ pkgs, ... }:

{
  imports = [
    ./hardware-configuration.nix

    ../../modules/common.nix
    ../../modules/desktop.nix
  ];

  networking.hostName = "desktop";

  time.timeZone = "Europe/Moscow";

  users.users.<ИМЯ_ПОЛЬЗОВАТЕЛЯ> = {
    isNormalUser = true;

    extraGroups = [
      "wheel"
      "networkmanager"
    ];
  };

  environment.systemPackages = with pkgs; [
    firefox
    neovim
  ];

  system.stateVersion = "25.05";
}
```

Замените:

```text
<ИМЯ_ПОЛЬЗОВАТЕЛЯ>
```

на имя пользователя.

Например:

```nix
users.users/alex = {
```

Правильный вариант:

```nix
users.users.alex = {
  isNormalUser = true;

  extraGroups = [
    "wheel"
    "networkmanager"
  ];
};
```

Необходимо также заменить:

```nix
system.stateVersion = "25.05";
```

на версию, которая использовалась при первоначальной установке NixOS.

Важно: `system.stateVersion` не нужно менять при каждом обновлении системы.

---

## Создание desktop-модуля

Создайте файл:

```bash
nano ~/nixos-config/modules/desktop.nix
```

Пример для GNOME:

```nix
{ pkgs, ... }:

{
  services.xserver.enable = true;

  services.displayManager.gdm.enable = true;

  services.desktopManager.gnome.enable = true;

  hardware.graphics.enable = true;

  environment.systemPackages = with pkgs; [
    firefox
    neovim
    alacritty
    file
    unzip
    zip
  ];
}
```

Если используется KDE Plasma, замените содержимое на:

```nix
{ pkgs, ... }:

{
  services.xserver.enable = true;

  services.displayManager.sddm.enable = true;

  services.desktopManager.plasma6.enable = true;

  hardware.graphics.enable = true;

  environment.systemPackages = with pkgs; [
    firefox
    neovim
    konsole
    file
    unzip
    zip
  ];
}
```

Если графическое окружение уже настроено в другой конфигурации, не нужно дублировать эти параметры.

---

# Создание конфигурации сервера

## Создание файла `hosts/server/default.nix`

Создайте файл:

```bash
nano ~/nixos-config/hosts/server/default.nix
```

Добавьте:

```nix
{ pkgs, ... }:

{
  imports = [
    ./hardware-configuration.nix

    ../../modules/common.nix
    ../../modules/server.nix
    ../../modules/ssh.nix
    ../../modules/firewall.nix
  ];

  networking.hostName = "server";

  time.timeZone = "Europe/Moscow";

  users.users.<ИМЯ_ПОЛЬЗОВАТЕЛЯ> = {
    isNormalUser = true;

    extraGroups = [
      "wheel"
    ];

    openssh.authorizedKeys.keys = [
      "<ПУБЛИЧНЫЙ_SSH_КЛЮЧ>"
    ];
  };

  environment.systemPackages = with pkgs; [
    vim
    git
    curl
    wget
  ];

  system.stateVersion = "25.05";
}
```

Замените:

```text
<ИМЯ_ПОЛЬЗОВАТЕЛЯ>
```

на имя пользователя сервера.

Замените:

```text
<ПУБЛИЧНЫЙ_SSH_КЛЮЧ>
```

на публичный SSH-ключ.

Пример:

```nix
users.users.admin = {
  isNormalUser = true;

  extraGroups = [
    "wheel"
  ];

  openssh.authorizedKeys.keys = [
    "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... user@desktop"
  ];
};
```

---

# Создание серверного модуля

Создайте файл:

```bash
nano ~/nixos-config/modules/server.nix
```

Добавьте:

```nix
{ ... }:

{
  services.openssh.enable = true;

  services.openssh.settings = {
    PermitRootLogin = "no";
    PasswordAuthentication = false;
    KbdInteractiveAuthentication = false;
  };
}
```

Этот модуль:

- включает SSH;
- запрещает вход под `root`;
- запрещает вход по паролю;
- оставляет авторизацию через SSH-ключи.

Перед отключением входа по паролю обязательно убедитесь, что вход по SSH-ключу работает.

---

# Создание SSH-модуля

Создайте файл:

```bash
nano ~/nixos-config/modules/ssh.nix
```

Добавьте:

```nix
{ ... }:

{
  services.openssh = {
    enable = true;

    settings = {
      PermitRootLogin = "no";
      PasswordAuthentication = false;
      KbdInteractiveAuthentication = false;
      X11Forwarding = false;
    };
  };
}
```

Если SSH уже настроен в `server.nix`, не обязательно подключать отдельный `ssh.nix`.

Не следует дублировать одни и те же параметры в нескольких модулях без необходимости.

---

# Создание Firewall-модуля

Создайте файл:

```bash
nano ~/nixos-config/modules/firewall.nix
```

Добавьте:

```nix
{ ... }:

{
  networking.firewall = {
    enable = true;

    allowedTCPPorts = [
      22
    ];
  };
}
```

В этом примере открыт только SSH-порт `22`.

Если нужно открыть HTTP и HTTPS:

```nix
{ ... }:

{
  networking.firewall = {
    enable = true;

    allowedTCPPorts = [
      22
      80
      443
    ];
  };
}
```

Если сервер использует Minecraft:

```nix
allowedTCPPorts = [
  22
  25565
];
```

Открывайте только те порты, которые действительно нужны.

---

# Создание hardware-конфигурации основной машины

На основной машине выполните:

```bash
sudo nixos-generate-config
```

В результате появятся файлы:

```text
/etc/nixos/configuration.nix
/etc/nixos/hardware-configuration.nix
```

Скопируйте hardware-конфигурацию в репозиторий:

```bash
sudo cp /etc/nixos/hardware-configuration.nix \
  ~/nixos-config/hosts/desktop/hardware-configuration.nix
```

Проверьте файл:

```bash
cat ~/nixos-config/hosts/desktop/hardware-configuration.nix
```

Файл должен содержать настройки дисков, файловых систем и оборудования именно основной машины.

---

# Создание hardware-конфигурации сервера

На сервере выполните:

```bash
sudo nixos-generate-config
```

Скопируйте файл в репозиторий:

```bash
sudo cp /etc/nixos/hardware-configuration.nix \
  ~/nixos-config/hosts/server/hardware-configuration.nix
```

Если репозиторий находится на другой машине, передайте файл через `scp`.

На сервере выполните:

```bash
sudo cat /etc/nixos/hardware-configuration.nix
```

Скопируйте содержимое.

На основной машине создайте файл:

```bash
nano ~/nixos-config/hosts/server/hardware-configuration.nix
```

Вставьте содержимое файла сервера.

Важно: `hardware-configuration.nix` основной машины нельзя использовать на сервере.

---

# Получение SSH-ключа

## Проверка существующего ключа

На основной машине выполните:

```bash
ls -la ~/.ssh
```

Если есть файл:

```text
id_ed25519.pub
```

выведите его содержимое:

```bash
cat ~/.ssh/id_ed25519.pub
```

Скопируйте весь вывод одной строкой.

---

## Создание нового SSH-ключа

Если ключа нет:

```bash
ssh-keygen -t ed25519
```

Можно нажать `Enter`, чтобы использовать стандартный путь:

```text
/home/<пользователь>/.ssh/id_ed25519
```

После создания вывести публичный ключ:

```bash
cat ~/.ssh/id_ed25519.pub
```

Добавьте его в:

```nix
openssh.authorizedKeys.keys = [
  "ssh-ed25519 AAAA... user@desktop"
];
```

---

# Первый запуск конфигурации

Перейдите в каталог репозитория:

```bash
cd ~/nixos-config
```

Проверьте структуру:

```bash
find . -maxdepth 3 -type f
```

Проверьте Flake:

```bash
nix flake check
```

Если `flake.lock` ещё не существует, Nix создаст его при первой сборке.

Соберите конфигурацию основной машины:

```bash
nix build .#nixosConfigurations.desktop.config.system.build.toplevel
```

Если сборка прошла успешно, примените конфигурацию:

```bash
sudo nixos-rebuild switch --flake .#desktop
```

---

# Применение конфигурации сервера

На сервере должен находиться этот же репозиторий.

Если репозиторий ещё не склонирован:

```bash
git clone https://github.com/<пользователь>/<репозиторий>.git ~/nixos-config
```

Перейдите в каталог:

```bash
cd ~/nixos-config
```

Проверьте конфигурацию:

```bash
nix flake check
```

Соберите конфигурацию сервера:

```bash
nix build .#nixosConfigurations.server.config.system.build.toplevel
```

Примените конфигурацию:

```bash
sudo nixos-rebuild switch --flake .#server
```

---

# Проверка конфигурации

## Проверка Flake

```bash
nix flake check
```

---

## Просмотр доступных конфигураций

```bash
nix flake show
```

В выводе должны быть конфигурации:

```text
nixosConfigurations
├── desktop
└── server
```

---

## Сборка без применения

Для основной машины:

```bash
nix build .#nixosConfigurations.desktop.config.system.build.toplevel
```

Для сервера:

```bash
nix build .#nixosConfigurations.server.config.system.build.toplevel
```

---

## Проверка результата сборки

После выполнения `nix build` появится ссылка:

```text
result
```

Посмотреть содержимое:

```bash
ls -la result
```

Удалить ссылку можно командой:

```bash
rm result
```

---

# Применение конфигурации

## Основная машина

```bash
cd ~/nixos-config
sudo nixos-rebuild switch --flake .#desktop
```

---

## Сервер

```bash
cd ~/nixos-config
sudo nixos-rebuild switch --flake .#server
```

---

## Временное тестирование

Команда `test` применяет конфигурацию только до следующей перезагрузки:

```bash
sudo nixos-rebuild test --flake .#desktop
```

Для сервера:

```bash
sudo nixos-rebuild test --flake .#server
```

---

## Применение при следующей загрузке

```bash
sudo nixos-rebuild boot --flake .#desktop
```

Для сервера:

```bash
sudo nixos-rebuild boot --flake .#server
```

---

# Скрипт `switch.sh`

Откройте файл:

```bash
nano ~/nixos-config/switch.sh
```

Добавьте:

```bash
#!/usr/bin/env bash

set -euo pipefail

HOST="${1:-}"

if [ -z "$HOST" ]; then
  echo "Не указана конфигурация."
  echo
  echo "Использование:"
  echo "  ./switch.sh desktop"
  echo "  ./switch.sh server"
  exit 1
fi

case "$HOST" in
  desktop|server)
    ;;
  *)
    echo "Неизвестная конфигурация: $HOST"
    echo "Доступные конфигурации: desktop, server"
    exit 1
    ;;
esac

sudo nixos-rebuild switch --flake ".#$HOST"
```

Сделайте файл исполняемым:

```bash
chmod +x ~/nixos-config/switch.sh
```

Применить конфигурацию основной машины:

```bash
./switch.sh desktop
```

Применить конфигурацию сервера:

```bash
./switch.sh server
```

---

# Первый коммит

Проверьте изменения:

```bash
cd ~/nixos-config
git status
```

Добавьте файлы:

```bash
git add .
```

Проверьте, что будет добавлено:

```bash
git diff --cached
```

Создайте коммит:

```bash
git commit -m "Add initial NixOS configurations"
```

Отправьте изменения:

```bash
git push -u origin main
```

Если основная ветка называется `master`:

```bash
git push -u origin master
```

Проверить название текущей ветки:

```bash
git branch --show-current
```

---

# Файл `.gitignore`

Откройте файл:

```bash
nano ~/nixos-config/.gitignore
```

Добавьте:

```gitignore
# Nix build results
result
result-*

# Direnv
.direnv/

# Environment files
.env
.env.*
!.env.example

# Secrets
secrets/*
!secrets/.gitkeep

# SSH private keys
id_rsa
id_rsa.pub
id_ed25519
id_ed25519.pub
*.pem
*.key

# Editors
.vscode/
.idea/
*.swp
*.swo
*~

# Operating system files
.DS_Store
Thumbs.db
```

Проверьте, что секреты не попадут в Git:

```bash
git status
```

---

# Работа с Git

## Просмотр состояния

```bash
git status
```

---

## Просмотр изменений

```bash
git diff
```

---

## Добавление изменений

Добавить конкретный файл:

```bash
git add hosts/desktop/default.nix
```

Добавить несколько файлов:

```bash
git add hosts/desktop/default.nix modules/desktop.nix
```

Добавить все изменения:

```bash
git add .
```

---

## Просмотр подготовленных изменений

```bash
git diff --cached
```

---

## Создание коммита

```bash
git commit -m "Update desktop configuration"
```

---

## Отправка изменений

```bash
git push
```

---

## Получение изменений

```bash
git pull
```

---

# Синхронизация основной машины и сервера

## Изменение конфигурации основной машины

На основной машине:

```bash
cd ~/nixos-config
```

Получить последние изменения:

```bash
git pull
```

Изменить нужные файлы.

Проверить:

```bash
nix flake check
```

Применить конфигурацию:

```bash
sudo nixos-rebuild switch --flake .#desktop
```

Сохранить изменения:

```bash
git add .
git commit -m "Update desktop configuration"
git push
```

---

## Обновление сервера

На сервере:

```bash
cd ~/nixos-config
```

Получить изменения:

```bash
git pull
```

Проверить:

```bash
nix flake check
```

Собрать конфигурацию:

```bash
nix build .#nixosConfigurations.server.config.system.build.toplevel
```

Применить:

```bash
sudo nixos-rebuild switch --flake .#server
```

---

# Рекомендуемый порядок обновления сервера

Перед изменением сервера:

```bash
ssh <пользователь>@<адрес-сервера>
```

Перейдите в репозиторий:

```bash
cd ~/nixos-config
```

Получите изменения:

```bash
git pull
```

Проверьте Flake:

```bash
nix flake check
```

Сначала выполните тестовое применение:

```bash
sudo nixos-rebuild test --flake .#server
```

Проверьте сервисы:

```bash
systemctl --failed
```

Если всё работает, примените конфигурацию постоянно:

```bash
sudo nixos-rebuild switch --flake .#server
```

После применения снова проверьте:

```bash
systemctl --failed
```

---

# Обновление зависимостей Flake

Обновить зависимости:

```bash
nix flake update
```

После обновления проверить конфигурацию:

```bash
nix flake check
```

Собрать конфигурацию основной машины:

```bash
nix build .#nixosConfigurations.desktop.config.system.build.toplevel
```

Собрать конфигурацию сервера:

```bash
nix build .#nixosConfigurations.server.config.system.build.toplevel
```

Если всё работает, применить конфигурацию:

```bash
sudo nixos-rebuild switch --flake .#desktop
```

Или:

```bash
sudo nixos-rebuild switch --flake .#server
```

Сохранить обновление:

```bash
git add flake.nix flake.lock
git commit -m "Update flake inputs"
git push
```

---

# Добавление пакетов

## Пакеты для обеих машин

Добавляйте общие пакеты в:

```text
modules/common.nix
```

Пример:

```nix
environment.systemPackages = with pkgs; [
  vim
  git
  curl
  wget
  htop
  tree
  tmux
  ripgrep
  fd
  jq
];
```

---

## Пакеты только для основной машины

Добавляйте пакеты в:

```text
hosts/desktop/default.nix
```

или:

```text
modules/desktop.nix
```

Пример:

```nix
environment.systemPackages = with pkgs; [
  firefox
  neovim
  discord
  vlc
];
```

---

## Пакеты только для сервера

Добавляйте пакеты в:

```text
hosts/server/default.nix
```

или:

```text
modules/server.nix
```

Пример:

```nix
environment.systemPackages = with pkgs; [
  vim
  tmux
  htop
  btop
  rsync
];
```

После изменения:

```bash
nix flake check
```

Применить:

```bash
sudo nixos-rebuild switch --flake .#desktop
```

или:

```bash
sudo nixos-rebuild switch --flake .#server
```

---

# Добавление сервисов

Сервисы лучше выносить в отдельные модули.

Например:

```text
modules/
├── common.nix
├── server.nix
├── nginx.nix
├── docker.nix
└── backup.nix
```

---

## Пример Nginx

Создайте файл:

```bash
nano ~/nixos-config/modules/nginx.nix
```

Добавьте:

```nix
{ ... }:

{
  services.nginx.enable = true;

  networking.firewall.allowedTCPPorts = [
    80
    443
  ];
}
```

Подключите модуль в:

```text
hosts/server/default.nix
```

Добавьте в `imports`:

```nix
imports = [
  ./hardware-configuration.nix

  ../../modules/common.nix
  ../../modules/server.nix
  ../../modules/ssh.nix
  ../../modules/firewall.nix
  ../../modules/nginx.nix
];
```

Проверить:

```bash
nix flake check
```

Применить:

```bash
sudo nixos-rebuild switch --flake .#server
```

---

## Пример PostgreSQL

Создайте файл:

```bash
nano ~/nixos-config/modules/postgresql.nix
```

Добавьте:

```nix
{ ... }:

{
  services.postgresql = {
    enable = true;

    ensureDatabases = [
      "app"
    ];

    ensureUsers = [
      {
        name = "app";
        ensureDBOwnership = true;
      }
    ];
  };

  networking.firewall.allowedTCPPorts = [
    5432
  ];
}
```

Если PostgreSQL используется только локально, порт `5432` лучше не открывать во внешнюю сеть.

В таком случае используйте:

```nix
{ ... }:

{
  services.postgresql = {
    enable = true;

    ensureDatabases = [
      "app"
    ];

    ensureUsers = [
      {
        name = "app";
        ensureDBOwnership = true;
      }
    ];
  };
}
```

---

# Настройка Docker

Создайте файл:

```bash
nano ~/nixos-config/modules/docker.nix
```

Добавьте:

```nix
{ ... }:

{
  virtualisation.docker.enable = true;
}
```

Подключите его в `hosts/server/default.nix`:

```nix
imports = [
  ./hardware-configuration.nix

  ../../modules/common.nix
  ../../modules/server.nix
  ../../modules/ssh.nix
  ../../modules/firewall.nix
  ../../modules/docker.nix
];
```

Добавьте пользователя в группу `docker`:

```nix
users.users.<ИМЯ_ПОЛЬЗОВАТЕЛЯ> = {
  isNormalUser = true;

  extraGroups = [
    "wheel"
    "docker"
  ];
};
```

Примените конфигурацию:

```bash
sudo nixos-rebuild switch --flake .#server
```

Проверьте Docker:

```bash
docker version
```

Если текущая сессия ещё не видит группу `docker`, выполните:

```bash
newgrp docker
```

Или выйдите из системы и войдите заново.

---

# Настройка Tailscale

Создайте файл:

```bash
nano ~/nixos-config/modules/tailscale.nix
```

Добавьте:

```nix
{ ... }:

{
  services.tailscale.enable = true;
}
```

Подключите модуль к серверу:

```nix
imports = [
  ./hardware-configuration.nix

  ../../modules/common.nix
  ../../modules/server.nix
  ../../modules/ssh.nix
  ../../modules/firewall.nix
  ../../modules/tailscale.nix
];
```

Примените конфигурацию:

```bash
sudo nixos-rebuild switch --flake .#server
```

Запустите авторизацию:

```bash
sudo tailscale up
```

---

# Настройка резервного копирования

Создайте файл:

```bash
nano ~/nixos-config/modules/backups.nix
```

Пример простого резервного копирования через `rsync`:

```nix
{ pkgs, ... }:

{
  environment.systemPackages = with pkgs; [
    rsync
  ];

  systemd.services.backup-home = {
    description = "Backup home directory";

    serviceConfig = {
      Type = "oneshot";
      User = "root";
    };

    script = ''
      ${pkgs.rsync}/bin/rsync \
        -a \
        --delete \
        /home/ \
        /backup/home/
    '';
  };

  systemd.timers.backup-home = {
    description = "Run home backup daily";

    wantedBy = [
      "timers.target"
    ];

    timerConfig = {
      OnCalendar = "daily";
      Persistent = true;
    };
  };
}
```

Перед использованием убедитесь, что каталог существует:

```bash
sudo mkdir -p /backup/home
```

Подключите модуль:

```nix
imports = [
  ./hardware-configuration.nix

  ../../modules/common.nix
  ../../modules/server.nix
  ../../modules/backups.nix
];
```

Проверить таймер:

```bash
systemctl list-timers
```

Проверить сервис:

```bash
systemctl status backup-home.service
```

---

# Работа с секретами

Не добавляйте в репозиторий секреты в открытом виде.

Нельзя хранить в Git:

```text
пароли
приватные SSH-ключи
API-токены
ключи доступа
.env-файлы
ключи шифрования
приватные сертификаты
пароли баз данных
```

Нельзя добавлять:

```text
id_rsa
id_ed25519
*.pem
*.key
.env
.env.production
passwords.txt
```

Для секретов используйте:

- `sops-nix`;
- `agenix`;
- Vault;
- переменные окружения;
- секреты, создаваемые вручную на сервере.

---

# Пример `.env.example`

Если приложению нужны переменные окружения, можно хранить только пример:

```text
DATABASE_URL=
SECRET_KEY=
API_TOKEN=
```

Файл можно назвать:

```text
.env.example
```

Настоящий `.env` должен быть добавлен в `.gitignore`.

---

# Откат конфигурации

## Быстрый откат

Если новая конфигурация сломала систему:

```bash
sudo nixos-rebuild switch --rollback
```

---

## Просмотр поколений

```bash
sudo nix-env --list-generations \
  --profile /nix/var/nix/profiles/system
```

---

## Просмотр текущей генерации

```bash
readlink /nix/var/nix/profiles/system
```

---

## Переключение на конкретную генерацию

Пример:

```bash
sudo /nix/var/nix/profiles/system-42-link/bin/switch-to-configuration switch
```

Замените `42` на нужный номер поколения.

---

## Откат через загрузчик

При загрузке NixOS можно выбрать предыдущую генерацию в меню загрузчика.

Если система перестала загружаться:

1. Перезагрузите компьютер.
2. Откройте меню загрузчика.
3. Выберите предыдущую генерацию NixOS.
4. Загрузите систему.
5. Исправьте конфигурацию в репозитории.
6. Выполните новую сборку.

---

# Очистка старых поколений

Посмотреть поколения:

```bash
sudo nix-env --list-generations \
  --profile /nix/var/nix/profiles/system
```

Удалить старые поколения`

Удалить все старые поколения:

```bash
sudo nix-collect-garbage -d
```

После очистки старые поколения будут недоступны для отката.

---

# Добавление новой машины

Допустим, нужно добавить машину с именем:

```text
laptop
```

Создайте каталог:

```bash
mkdir -p hosts/laptop
```

На новой машине создайте hardware-конфигурацию:

```bash
sudo nixos-generate-config
```

Скопируйте файл:

```bash
cp /etc/nixos/hardware-configuration.nix \
  hosts/laptop/hardware-configuration.nix
```

Создайте файл:

```bash
nano hosts/laptop/default.nix
```

Добавьте:

```nix
{ pkgs, ... }:

{
  imports = [
    ./hardware-configuration.nix
    ../../modules/common.nix
    ../../modules/desktop.nix
  ];

  networking.hostName = "laptop";

  time.timeZone = "Europe/Moscow";

  users.users.<ИМЯ_ПОЛЬЗОВАТЕЛЯ> = {
    isNormalUser = true;

    extraGroups = [
      "wheel"
      "networkmanager"
    ];
  };

  system.stateVersion = "25.05";
}
```

Добавьте новую машину в `flake.nix`:

```nix
laptop = nixpkgs.lib.nixosSystem {
  system = "x86_64-linux";

  modules = [
    ./hosts/laptop
  ];
};
```

Итоговый фрагмент `flake.nix`:

```nix
nixosConfigurations = {
  desktop = nixpkgs.lib.nixosSystem {
    system = "x86_64-linux";

    modules = [
      ./hosts/desktop
    ];
  };

  server = nixpkgs.lib.nixosSystem {
    system = "x86_64-linux";

    modules = [
      ./hosts/server
    ];
  };

  laptop = nixpkgs.lib.nixosSystem {
    system = "x86_64-linux";

    modules = [
      ./hosts/laptop
    ];
  };
};
```

Проверить:

```bash
nix flake check
```

Применить:

```bash
sudo nixos-rebuild switch --flake .#laptop
```

---

# Переименование машины

Если нужно переименовать `desktop` в `main-pc`, измените:

```text
hosts/desktop/
```

на:

```text
hosts/main-pc/
```

Переименуйте конфигурацию в `flake.nix`:

```nix
main-pc = nixpkgs.lib.nixosSystem {
  system = "x86_64-linux";

  modules = [
    ./hosts/main-pc
  ];
};
```

Измените hostname:

```nix
networking.hostName = "main-pc";
```

После этого используйте:

```bash
sudo nixos-rebuild switch --flake .#main-pc
```

---

# Подключение Home Manager

Home Manager используется для управления пользовательскими настройками.

Добавьте `home-manager` в `flake.nix`:

```nix
{
  description = "NixOS configurations";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";

    home-manager = {
      url = "github:nix-community/home-manager";

      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = { self, nixpkgs, home-manager, ... }:
    {
      nixosConfigurations = {
        desktop = nixpkgs.lib.nixosSystem {
          system = "x86_64-linux";

          modules = [
            ./hosts/desktop

            home-manager.nixosModules.home-manager

            {
              home-manager.useGlobalPkgs = true;
              home-manager.useUserPackages = true;

              home-manager.users.<ИМЯ_ПОЛЬЗОВАТЕЛЯ> =
                import ./home/desktop.nix;
            }
          ];
        };

        server = nixpkgs.lib.nixosSystem {
          system = "x86_64-linux";

          modules = [
            ./hosts/server

            home-manager.nixosModules.home-manager

            {
              home-manager.useGlobalPkgs = true;
              home-manager.useUserPackages = true;

              home-manager.users.<ИМЯ_ПОЛЬЗОВАТЕЛЯ> =
                import ./home/server.nix;
            }
          ];
        };
      };
    };
}
```

---

## Файл `home/desktop.nix`

Создайте:

```bash
nano ~/nixos-config/home/desktop.nix
```

Добавьте:

```nix
{ pkgs, ... }:

{
  home.username = "<ИМЯ_ПОЛЬЗОВАТЕЛЯ>";
  home.homeDirectory = "/home/<ИМЯ_ПОЛЬЗОВАТЕЛЯ>";

  home.packages = with pkgs; [
    ripgrep
    fd
    jq
    btop
  ];

  programs.git = {
    enable = true;

    userName = "Ваше имя";
    userEmail = "your@email.com";
  };

  programs.bash.enable = true;

  home.stateVersion = "25.05";
}
```

---

## Файл `home/server.nix`

Создайте:

```bash
nano ~/nixos-config/home/server.nix
```

Добавьте:

```nix
{ pkgs, ... }:

{
  home.username = "<ИМЯ_ПОЛЬЗОВАТЕЛЯ>";
  home.homeDirectory = "/home/<ИМЯ_ПОЛЬЗОВАТЕЛЯ>";

  home.packages = with pkgs; [
    btop
    ripgrep
    fd
    jq
  ];

  programs.git = {
    enable = true;

    userName = "Ваше имя";
    userEmail = "your@email.com";
  };

  programs.bash.enable = true;

  home.stateVersion = "25.05";
}
```

---

# Проверка сервисов

После применения конфигурации проверьте систему:

```bash
systemctl --failed
```

Проверить SSH:

```bash
systemctl status sshd
```

Проверить Docker:

```bash
systemctl status docker
```

Проверить Nginx:

```bash
systemctl status nginx
```

Посмотреть последние ошибки:

```bash
journalctl -p err -b
```

Посмотреть логи конкретного сервиса:

```bash
journalctl -u sshd
```

```bash
journalctl -u docker
```

```bash
journalctl -u nginx
```

---

# Полный порядок действий с нуля

## На первой машине

Создайте каталог:

```bash
mkdir -p ~/nixos-config
cd ~/nixos-config
```

Инициализируйте Git:

```bash
git init
```

Создайте структуру:

```bash
mkdir -p hosts/desktop
mkdir -p hosts/server
mkdir -p modules
mkdir -p home
mkdir -p secrets
touch secrets/.gitkeep
```

Создайте файлы:

```bash
touch flake.nix
touch .gitignore
touch switch.sh
touch README.md
```

Вставьте содержимое в:

```text
flake.nix
modules/common.nix
modules/desktop.nix
modules/server.nix
modules/ssh.nix
modules/firewall.nix
hosts/desktop/default.nix
hosts/server/default.nix
.gitignore
switch.sh
```

Создайте hardware-конфигурацию основной машины:

```bash
sudo nixos-generate-config
```

Скопируйте её:

```bash
sudo cp /etc/nixos/hardware-configuration.nix \
  hosts/desktop/hardware-configuration.nix
```

Проверьте:

```bash
nix flake check
```

Соберите конфигурацию:

```bash
nix build .#nixosConfigurations.desktop.config.system.build.toplevel
```

Примените:

```bash
sudo nixos-rebuild switch --flake .#desktop
```

Сохраните в Git:

```bash
git add .
git commit -m "Add initial NixOS configuration"
```

Отправьте в GitHub:

```bash
git remote add origin https://github.com/<пользователь>/<репозиторий>.git
git branch -M main
git push -u origin main
```

---

## На сервере

Клонируйте репозиторий:

```bash
git clone https://github.com/<пользователь>/<репозиторий>.git ~/nixos-config
```

Перейдите в каталог:

```bash
cd ~/nixos-config
```

Создайте hardware-конфигурацию:

```bash
sudo nixos-generate-config
```

Скопируйте её:

```bash
sudo cp /etc/nixos/hardware-configuration.nix \
  hosts/server/hardware-configuration.nix
```

Проверьте конфигурацию:

```bash
nix flake check
```

Соберите серверную конфигурацию:

```bash
nix build .#nixosConfigurations.server.config.system.build.toplevel
```

Примените её:

```bash
sudo nixos-rebuild switch --flake .#server
```

После успешного применения сохраните hardware-конфигурацию:

```bash
git add hosts/server/hardware-configuration.nix
git commit -m "Add server hardware configuration"
git push
```

---

# Рекомендуемый рабочий процесс

Каждый раз перед изменением конфигурации:

```bash
cd ~/nixos-config
git pull
```

Внесите изменения.

Проверьте Flake:

```bash
nix flake check
```

Соберите нужную конфигурацию:

```bash
nix build .#nixosConfigurations.desktop.config.system.build.toplevel
```

или:

```bash
nix build .#nixosConfigurations.server.config.system.build.toplevel
```

Примените конфигурацию:

```bash
sudo nixos-rebuild switch --flake .#desktop
```

или:

```bash
sudo nixos-rebuild switch --flake .#server
```

Проверьте состояние системы:

```bash
systemctl --failed
```

Проверьте Git:

```bash
git status
```

Посмотрите изменения:

```bash
git diff
```

Добавьте изменения:

```bash
git add .
```

Создайте коммит:

```bash
git commit -m "Describe configuration change"
```

Отправьте изменения:

```bash
git push
```

---

# Пример сообщений коммитов

```text
Add initial desktop configuration
Add server configuration
Configure SSH access
Enable Docker on server
Add firewall rules
Enable GNOME desktop
Add Nginx service
Add PostgreSQL service
Configure automatic garbage collection
Update nixpkgs input
Add backup service
Fix server hostname
Update desktop packages
```

---

# Итоговые команды

## Основная машина

```bash
cd ~/nixos-config
git pull
nix flake check
sudo nixos-rebuild switch --flake .#desktop
git add .
git commit -m "Update desktop configuration"
git push
```

## Сервер

```bash
cd ~/nixos-config
git pull
nix flake check
sudo nixos-rebuild switch --flake .#server
git add .
git commit -m "Update server configuration"
git push
```

## Откат

```bash
sudo nixos-rebuild switch --rollback
```

## Обновление зависимостей

```bash
nix flake update
nix flake check
sudo nixos-rebuild switch --flake .#desktop
```

## Добавление новой машины

```bash
mkdir -p hosts/<имя-машины>
```

Добавить конфигурацию машины в:

```text
hosts/<имя-машины>/default.nix
```

Добавить hardware-конфигурацию:

```text
hosts/<имя-машины>/hardware-configuration.nix
```

Добавить машину в:

```text
flake.nix
```

Проверить:

```bash
nix flake check
```

Применить:

```bash
sudo nixos-rebuild switch --flake .#<имя-машины>
```