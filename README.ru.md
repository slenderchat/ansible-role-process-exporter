# ansible-role-process-exporter

[English](README.md) | [Русский](README.ru.md)

Ansible-роль устанавливает [Process Exporter](https://github.com/ncabatoff/process-exporter) из системного RPM-пакета и запускает его как защищённый сервис systemd. Сервис работает от отдельного системного пользователя с двумя Linux capabilities: `CAP_DAC_READ_SEARCH` и `CAP_SYS_PTRACE`.

Автор реализации — Nikita Grabar.

> **Авторские права.** Copyright (c) 2026 Nikita Grabar. All rights reserved. Это ПО не распространяется под лицензией open source. Использование, копирование, изменение и распространение требуют предварительного разрешения правообладателя. См. [COPYRIGHT](COPYRIGHT).

## Поддерживаемые платформы

- РЕД ОС 7.3: `golang-github-ncabatoff-process-exporter-0.8.7-1.el7`.
- РЕД ОС 8: `golang-github-ncabatoff-process-exporter-0.8.7-1.red80`.

Для обеих ОС используется одинаковая реализация. Отдельный пакет `process-exporter`, доступный в РЕД ОС 8, не применяется.

Роль проверена на РЕД ОС 7.3 и 8 как при наличии RPM-пакета, так и при установке с нуля. Успешны первое выполнение, повторный запуск (`changed=0`, `failed=0`) и финальный `--check`. На работающих узлах подтверждены включённый и активный сервис, пользователь, маски capabilities, `NoNewPrivs=1`, единственный listener на `127.0.0.1:9256`, HTTP 200 для `/metrics` и отсутствие предупреждений systemd journal. Приём данных внешним сборщиком этими проверками не подтверждается.

## Требования

- Пакет `golang-github-ncabatoff-process-exporter` доступен в подключённых репозиториях ОС. Используется `state: present`: версия не фиксируется, а уже установленный пакет роль автоматически не обновляет.
- Systemd с поддержкой `AmbientCapabilities=` (версия 229 и выше).
- Команда `ss` (пакет `iproute`) для проверок после развёртывания.
- Повышение привилегий (`become: true`) для установки пакета, создания пользователя и управления сервисом.
- Достаточно модулей `ansible.builtin`; сторонние коллекции не нужны.
- Порт `127.0.0.1:9256` должен быть свободен при запуске роли. Если его занимает Docker-контейнер, см. раздел «Миграция с Docker и откат».

## Установка

Разместите содержимое репозитория в каталоге `process_exporter` одного из путей поиска Ansible-ролей, например `roles/process_exporter/` рядом с вашими playbook, либо укажите родительский каталог через `roles_path` или `ANSIBLE_ROLES_PATH`. Зависимостей от других ролей нет.

Пример:

```bash
mkdir -p roles
git clone https://github.com/slenderchat/ansible-role-process-exporter.git roles/process_exporter
```

## Конфигурация

Настраиваемых переменных роли нет. Адрес `127.0.0.1:9256`, пути файлов и набор capabilities намеренно фиксированы.

### Группировка процессов

```yaml
process_names:
  - name: "{{.Comm}}"
    cmdline:
      - ".+"
```

Группа соответствует имени команды (`comm`, максимум 15 символов). Процессы с пустой командной строкой (в частности, потоки ядра) не учитываются. Остальные опции экспортера сохраняют значения по умолчанию: включены `-threads`, `-gather-smaps`, `-children`, а procfs читается из `/proc`.

## Использование

Пример пользовательского playbook `site.yml`:

```yaml
---
- name: Deploy Process Exporter
  hosts: all
  become: true
  gather_facts: false
  roles:
    - process_exporter
```

Сначала проверьте изменения в `--check`, затем выполните развёртывание, ограничив список хостов:

```bash
ANSIBLE_ROLES_PATH=./roles ansible-playbook -i INVENTORY site.yml -l HOST --check --diff
ANSIBLE_ROLES_PATH=./roles ansible-playbook -i INVENTORY site.yml -l HOST --diff
```

### Действия роли

1. Устанавливает RPM-пакет, содержащий `/usr/bin/process-exporter`, но не содержащий unit и конфигурацию.
2. Создаёт системную группу и пользователя `process-exporter` с `/sbin/nologin`, домашним путём `/`, без создания домашнего каталога.
3. Создаёт конфигурацию `/opt/process-exporter/process-exporter.yml` и unit `/etc/systemd/system/process-exporter.service`; оба файла принадлежат `root:root`, режим `0644`.
4. Удаляет устаревший `/etc/sysconfig/process-exporter` только после проверки, что systemd загрузил новый unit. Старый unit использовал из этого файла `OPTIONS` и без него мог начать слушать все интерфейсы. Если проверка не проходит, выполнение прерывается, а файл сохраняется.
5. Включает автозапуск и запускает сервис `process-exporter`.
6. Проверяет сервис, listener и выдачу метрик.

Изменение конфигурации вызывает restart; изменение unit — `daemon-reload` и restart. Без изменений лишних перезапусков нет. Роль дополнительно проверяет `NeedDaemonReload`: если предыдущий запуск успел записать unit, но прервался перед `daemon-reload`, следующий запуск завершит обновление.

## Привилегии и безопасность

Сервис запускается пользователем `process-exporter` со строго двумя capabilities:

```ini
CapabilityBoundingSet=CAP_DAC_READ_SEARCH CAP_SYS_PTRACE
AmbientCapabilities=CAP_DAC_READ_SEARCH CAP_SYS_PTRACE
NoNewPrivileges=yes
```

`AmbientCapabilities=` предоставляет права процессу непривилегированного пользователя. `CapabilityBoundingSet=` запрещает получение иных capabilities, а `NoNewPrivileges=yes` блокирует повышение привилегий через setuid и файловые capabilities. У работающего процесса `CapInh`, `CapPrm`, `CapEff`, `CapBnd` и `CapAmb` равны `0000000000080004`.

Применяются дополнительные ограничения systemd, в том числе `ProtectSystem=strict`, `ProtectHome`, `PrivateTmp`, `PrivateDevices`, `ProtectKernel*`, `RestrictAddressFamilies` и другие. Ограничения видимости `/proc` (`ProtectProc`, `ProcSubset`, `PrivateUsers`) намеренно не используются. Набор capabilities не настраивается переменными роли.

### Почему нужны обе capabilities

Без capabilities пользователь `process-exporter` может читать общедоступные сведения из `/proc/<pid>/stat`, `status`, `cmdline` и `limits`: число процессов и потоков, CPU, page faults, context switches, основную статистику памяти, состояния и время запуска. Для метрик чужих процессов, в том числе принадлежащих root, требуются дополнительные права:

| Источник | Метрики `namedprocess_namegroup_*` | Необходимая capability |
|---|---|---|
| `/proc/<pid>/io` | `read_bytes_total`, `write_bytes_total`, `thread_io_bytes_total` | `CAP_SYS_PTRACE` |
| `/proc/<pid>/smaps_rollup` | `memory_bytes` с `proportionalResident` и `proportionalSwapped` | `CAP_SYS_PTRACE` |
| `/proc/<pid>/fd` | `open_filedesc`, `worst_fd_ratio` | `CAP_DAC_READ_SEARCH` |

На РЕД ОС 7.3 и 8 проверялись временные процессы экспортера без capabilities, с каждой из двух по отдельности и с обеими вместе. Только совместное предоставление прав восстановило проверенные метрики процессов других пользователей и устранило постоянный рост `namedprocess_scrape_partial_errors`. Маски прав проверялись у реально запущенного процесса.

`smaps_rollup` появился в Linux 4.14; на более ранних ядрах соответствующая статистика недоступна независимо от привилегий.

### Последствия для безопасности

Это **не** режим с низкими привилегиями. Компрометацию экспортера с указанными capabilities необходимо рассматривать как потенциальную компрометацию всего хоста.

- `CAP_SYS_PTRACE` позволяет обходить проверки доступа при ptrace, включая процессы root, читать `/proc/<pid>/environ` и память процессов, подключаться через `ptrace(2)` и изменять память. Значения `kernel.yama.ptrace_scope` 1 и 2 не ограничивают обладателя этой capability. Захват экспортера может привести к выполнению кода с правами root.
- `CAP_DAC_READ_SEARCH` позволяет обходить проверки чтения файлов и поиска в каталогах: доступны видимые процессу файлы, включая `/etc/shadow`, ключи и конфигурации с паролями. `ProtectHome` и `PrivateTmp` скрывают пути, но не образуют надёжную границу безопасности: capability также допускает `open_by_handle_at(2)`.
- Ограничения: нельзя получить другие capabilities; не предоставлена `CAP_DAC_OVERRIDE`; `ProtectSystem=strict` делает файловую систему доступной только для чтения; сервис слушает только localhost; у пользователя нет интерактивной оболочки.
- В сравнении с привилегированным контейнером root набор прав уже, но по-прежнему чувствителен. Исправления безопасности поступают с обновлениями пакетов ОС; роль сама пакет не обновляет.

Не запускайте сервис от root и не расширяйте набор capabilities. Удаление любой из двух capabilities приведёт к недоступности части перечисленных метрик чужих процессов.

## Проверка работоспособности

После реального запуска, но не в check mode, роль:

1. Ожидает ответа `/metrics` по `http://127.0.0.1:9256/metrics`.
2. Проверяет активность сервиса и то, что единственный listener порта `9256` находится на `127.0.0.1:9256` и принадлежит `MainPID` этого сервиса: одного HTTP 200 недостаточно, поскольку ответить может оставшийся Docker-контейнер.
3. Проверяет группы `systemd` (PID 1, root) и `process-exporte` (имя процесса экспортера в `comm`).

Счётчики `namedprocess_scrape_*` роль не проверяет: они накопительные, а кратковременные partial errors могут возникать и в контейнерном, и в нативном варианте. Маски capabilities сама роль также не сверяет. Успешные проверки не гарантируют полноту метрик каждого процесса.

Ручная проверка на хосте:

```bash
systemctl is-enabled process-exporter
systemctl is-active process-exporter
systemctl show process-exporter -p NeedDaemonReload -p User -p ExecStart
ss -ltnp 'sport = :9256'
PID=$(systemctl show -p MainPID process-exporter | cut -d= -f2)
ps -o user=,comm= -p "$PID"
grep -E '^(CapInh|CapPrm|CapEff|CapBnd|CapAmb|NoNewPrivs)' "/proc/$PID/status"
curl --noproxy '*' -fsS http://127.0.0.1:9256/metrics \
  | grep -E '^namedprocess_namegroup_(open_filedesc|memory_bytes)[{]groupname="systemd"'
```

Ожидаются `enabled`, `active`, `NeedDaemonReload=no`, listener только `127.0.0.1:9256`, маски `0000000000080004`, `NoNewPrivs: 1`. Для группы `systemd` показатели файловых дескрипторов и пропорциональной резидентной памяти должны быть больше нуля. Убедитесь отдельно, что ваша система мониторинга получает метрики.

## Миграция с Docker и откат

Если контейнер `process-exporter` уже занимает `127.0.0.1:9256`, нативный сервис не сможет запуститься (`address already in use`). Systemd будет перезапускать его каждые 5 секунд в соответствии с `Restart=on-failure`, а роль завершится ошибкой проверки listener. Docker управляется вручную; роль не останавливает и не удаляет контейнеры. Переключайтесь по одному хосту в согласованное окно: между остановкой контейнера и запуском сервиса метрики не собираются, а после переключения экспортируемые счётчики начинают отсчёт заново.

1. **Подготовка без изменения работающего контейнера:** сохраните `docker inspect process-exporter > /root/process-exporter.inspect.json`, зафиксируйте политику перезапуска `docker inspect -f '{{.HostConfig.RestartPolicy.Name}}' process-exporter`, отключите стороннюю автоматизацию, которая может восстановить контейнер. Запустите роль с `--check --diff` и убедитесь в отсутствии ошибок.
2. **Остановите контейнер, не удаляя его:**

   ```bash
   docker update --restart=no process-exporter
   docker stop process-exporter
   ss -ltnp 'sport = :9256'
   ```

   Порт должен быть свободен. При другом имени контейнера замените `process-exporter` в Docker-командах.
3. **Запустите роль без `--check`**, ограничив выполнение одним целевым хостом. Дождитесь успешной проверки listener, сервиса и `/metrics`.
4. **Проверьте** маски capabilities, HTTP-ответ, метрики чужих процессов и получение данных вашей системой мониторинга (см. предыдущий раздел).
5. **Завершите миграцию:** только после принятия результата удалите остановленный контейнер `docker rm process-exporter`. Решение об удалении Docker-образа принимается отдельно.

Если роль ошибочно запущена при занятом порте, контейнер продолжит обслуживать метрики, а нативный сервис останется включённым и будет пытаться перезапускаться. Верните исходное состояние либо выполните переключение и повторите запуск роли.

### Откат

Пока контейнер сохранён:

```bash
systemctl disable --now process-exporter
docker update --restart=POLICY process-exporter
docker start process-exporter
ss -ltnp 'sport = :9256'
```

Вместо `POLICY` укажите ранее сохранённую restart policy (включая лимит попыток, если он был настроен). Сначала освобождайте порт остановкой systemd-сервиса. После отката **не** запускайте роль на этом хосте до следующего переключения: она снова включит нативный сервис. Если контейнер уже удалён, для отката потребуется его восстановление по сохранённому `docker inspect`.

## Режим проверки (`--check`)

Роль поддерживает `--check` и `--diff` на хостах как с установленным пакетом, так и без него. Если установка RPM или unit только моделируется, действия по запуску, включению автозапуска и перезапуску сервиса пропускаются. Проверки `/metrics` в check mode не выполняются.

## Мониторинг

Экспортер слушает только `127.0.0.1:9256`, поэтому опрашивать его можно только локально, например агентом мониторинга на том же узле. Роль не настраивает Prometheus, VictoriaMetrics или иные системы сбора данных.

## Теги

Отсутствуют.

## Авторские права

Copyright (c) 2026 Nikita Grabar. All rights reserved. Разрешение на использование не предоставляется автоматически; см. [COPYRIGHT](COPYRIGHT).
