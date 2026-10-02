# LightHouse playbook

Ansible-плейбук для установки и настройки веб-интерфейса [LightHouse](https://github.com/VKCOM/lighthouse) (UI для ClickHouse) на выделенном хосте.

## Что делает плейбук

Playbook `site.yml` выполняется для группы хостов `lighthouse` и:

1. Устанавливает `nginx` (с предварительным обновлением кэша пакетов).
2. Создаёт директорию для статики LightHouse (`{{ lighthouse_root }}`) с владельцем `www-data`.
3. Скачивает архив исходников LightHouse по ссылке `{{ lighthouse_src_url }}` во временный файл `{{ lighthouse_archive_path }}`.
4. Распаковывает архив в директорию статики (со срезанием корневой папки архива).
5. Генерирует конфиг nginx-сайта из шаблона `templates/lighthouse_nginx.conf.j2` и подключает его в `sites-enabled`.
6. Отключает дефолтный сайт nginx.
7. Запускает и включает автозапуск `nginx`.

При изменении конфигурации nginx (шаги 5–6) срабатывает хендлер `Reload nginx`, который перезагружает сервис.

## Структура

```
playbook/
├── ansible.cfg              # инвентарь по умолчанию, отключение host key checking
├── site.yml                 # сам playbook
├── inventory/
│   └── prod.yml             # хосты: clickhouse, vector, lighthouse
├── group_vars/
│   ├── all.yml               # общие параметры подключения для всех групп
│   └── lighthouse.yml        # параметры, специфичные для группы lighthouse
└── templates/
    └── lighthouse_nginx.conf.j2  # шаблон vhost-конфига nginx
```

## Параметры (переменные)

### `group_vars/all.yml` — общие для всех хостов

| Переменная | Значение по умолчанию | Описание |
|---|---|---|
| `ansible_user` | `ubuntu` | пользователь для SSH-подключения |
| `ansible_ssh_private_key_file` | `~/.ssh/id_ed25519` | приватный ключ для подключения |
| `ansible_python_interpreter` | `/usr/bin/python3` | путь к интерпретатору Python на хосте |

### `group_vars/lighthouse.yml` — специфичные для хоста LightHouse

| Переменная | Значение по умолчанию | Описание |
|---|---|---|
| `lighthouse_version` | `master` | ветка/тег репозитория LightHouse, которую нужно скачать |
| `lighthouse_src_url` | `https://github.com/VKCOM/lighthouse/archive/refs/heads/{{ lighthouse_version }}.tar.gz` | ссылка на архив исходников |
| `lighthouse_archive_path` | `/tmp/lighthouse.tar.gz` | путь для скачанного архива на хосте |
| `lighthouse_root` | `/var/www/lighthouse` | директория, куда распаковывается статика и откуда её отдаёт nginx |
| `lighthouse_nginx_port` | `80` | порт, который слушает nginx |
| `lighthouse_nginx_server_name` | `_` | значение `server_name` в конфиге nginx |

Переопределить любой параметр можно через `-e`, например:

```bash
ansible-playbook site.yml -e lighthouse_version=v1.4.0 -e lighthouse_nginx_port=8080
```

## Теги

| Тег | Что выполняет |
|---|---|
| `packages` | установка nginx |
| `deploy` | создание директории, скачивание и распаковка архива LightHouse |
| `nginx` | конфигурация и (пере)запуск nginx: шаблон, enable/disable сайтов, systemd |

Примеры запуска с тегами:

```bash
# только обновить статику LightHouse, без трогания nginx
ansible-playbook site.yml --tags deploy

# пересобрать только конфиг nginx
ansible-playbook site.yml --tags nginx

# пропустить установку пакетов (например, nginx уже стоит)
ansible-playbook site.yml --skip-tags packages
```

## Запуск

```bash
# проверка синтаксиса
ansible-playbook site.yml --syntax-check

# dry-run с выводом диффа
ansible-playbook site.yml --check --diff

# полный прогон
ansible-playbook site.yml

# прогон только по хосту/группе lighthouse
ansible-playbook site.yml --limit lighthouse
```

Инвентарь по умолчанию берётся из `inventory/prod.yml` (настроено в `ansible.cfg`), группа `lighthouse` содержит хост с итоговым дашбордом.
