# AWG 3.1 через российский VPS: UDP-реле на FRP

Пошаговое руководство для случая, когда прямой входящий UDP от иностранного VPS до домашнего подключения фильтруется, но UDP между домом и российским VPS работает.

## Содержание

- [1. Что строим](#1-что-строим)
- [2. Распределение ролей](#2-распределение-ролей)
- [3. Установка FRP на оба VPS](#3-установка-frp-на-оба-vps)
  - [3.1. Проверить архитектуру](#31-проверить-архитектуру)
  - [3.2. Установка для x86_64](#32-установка-для-x86_64)
  - [3.3. Установка для ARM64](#33-установка-для-arm64)
  - [3.4. Проверка установки](#34-проверка-установки)
  - [3.5. Создать системного пользователя](#35-создать-системного-пользователя)
- [4. Создать общий токен](#4-создать-общий-токен)
- [5. Настройка российского VPS: `frps`](#5-настройка-российского-vps-frps)
  - [5.1. Создать конфигурацию сервера](#51-создать-конфигурацию-сервера)
  - [5.2. Создать службу `frps`](#52-создать-службу-frps)
  - [5.3. Настроить и включить UFW](#53-настроить-и-включить-ufw)
    - [5.3.1. Проверить SSH-порт](#531-проверить-ssh-порт)
    - [5.3.2. Задать политики и разрешить нужные соединения](#532-задать-политики-и-разрешить-нужные-соединения)
    - [5.3.3. Проверить правила и включить UFW](#533-проверить-правила-и-включить-ufw)
    - [5.3.4. Проверить SSH из второго терминала](#534-проверить-ssh-из-второго-терминала)
- [6. Настройка иностранного VPS: `frpc`](#6-настройка-иностранного-vps-frpc)
  - [6.1. Проверить доступность российского `frps`](#61-проверить-доступность-российского-frps)
  - [6.2. Проверить локальный AWG-сервер](#62-проверить-локальный-awg-сервер)
  - [6.3. Создать конфигурацию клиента](#63-создать-конфигурацию-клиента)
  - [6.4. Создать службу `frpc`](#64-создать-службу-frpc)
- [7. Проверить регистрацию UDP-прокси](#7-проверить-регистрацию-udp-прокси)
- [8. Изменить AWG на Keenetic](#8-изменить-awg-на-keenetic)
- [9. Проверить работу всего маршрута](#9-проверить-работу-всего-маршрута)
  - [9.1. На российском VPS](#91-на-российском-vps)
  - [9.2. На иностранном VPS](#92-на-иностранном-vps)
  - [9.3. На Keenetic](#93-на-keenetic)
- [10. Если FRP подключён, но AWG-handshake отсутствует](#10-если-frp-подключён-но-awg-handshake-отсутствует)
- [11. Добавить второй иностранный AWG-сервер](#11-добавить-второй-иностранный-awg-сервер)
  - [11.1. Российский VPS](#111-российский-vps)
  - [11.2. Второй иностранный VPS](#112-второй-иностранный-vps)
- [12. Повседневное обслуживание](#12-повседневное-обслуживание)
  - [Статус служб](#статус-служб)
  - [Последние логи](#последние-логи)
  - [Перезапуск](#перезапуск)
  - [Проверка портов](#проверка-портов)
- [13. Что важно помнить](#13-что-важно-помнить)
- [14. Замечание о TLS](#14-замечание-о-tls)
- [15. Официальные источники](#15-официальные-источники)


## 1. Что строим

```text
Keenetic
  Endpoint = 170.168.0.00:12345
          │
          │ UDP внутри РФ
          ▼
Российский VPS 170.168.0.00
  frps: 7443/tcp (TCP) или 7443/udp (QUIC)
  UDP-реле: 12345/udp
          │
          │ FRP поверх TCP/TLS или QUIC/UDP
          ▼
Иностранный VPS freedom
  frpc → 127.0.0.1:12345/udp
          │
          ▼
Docker-контейнер amnezia-awg2
```

Российский VPS служит только транспортным реле. Он не получает ключи AWG и не расшифровывает содержимое AWG-туннеля. Выходной IP для трафика остаётся IP иностранного VPS.

## 2. Распределение ролей

| Узел | Адрес/имя | Что устанавливается |
|---|---|---|
| Российский VPS | `170.168.0.00` | `frps`, `/etc/frp/frps.toml`, `frps.service` |
| Иностранный VPS | `freedom`, публичный IP `179.255.000.000` | `frpc`, `/etc/frp/frpc.toml`, `frpc.service` |
| Keenetic | домашний роутер | FRP не устанавливается; меняется только AWG `Endpoint` |

Используемые порты:

| Порт | Где | Назначение |
|---|---|---|
| `7443/tcp` | российский VPS | соединение `frpc → frps` при выборе TCP |
| `7443/udp` | российский VPS | соединение `frpc → frps` при выборе QUIC (опционально) |
| `12345/udp` | российский VPS | адрес, к которому подключается Keenetic |
| `12345/udp` | иностранный VPS | локальный порт существующего AWG-сервера |

## 3. Установка FRP на оба VPS

Этот раздел выполнить **дважды**:

1. на российском VPS;
2. на иностранном VPS `freedom`.

В руководстве зафиксирована версия FRP `0.71.0`, актуальная на момент настройки 8 октября 2026 года.

### 3.1. Проверить архитектуру

```bash
uname -m
```

- `x86_64` — использовать сборку `linux_amd64`;
- `aarch64` или `arm64` — использовать сборку `linux_arm64`.

### 3.2. Установка для x86_64

```bash
cd /tmp

curl -fLO \
  https://github.com/fatedier/frp/releases/download/v0.71.0/frp_0.71.0_linux_amd64.tar.gz
```

Проверить архив:

```bash
echo '84f27e39f11169f7adcef8e8b70c9329de17747b1f14dad9fb95eef5682ea716  frp_0.71.0_linux_amd64.tar.gz' \
  | sha256sum -c -
```

Ожидаемый результат:

```text
frp_0.71.0_linux_amd64.tar.gz: OK
```

Установить бинарники:

```bash
tar -xzf frp_0.71.0_linux_amd64.tar.gz

sudo install -m 0755 \
  frp_0.71.0_linux_amd64/frps \
  /usr/local/bin/frps

sudo install -m 0755 \
  frp_0.71.0_linux_amd64/frpc \
  /usr/local/bin/frpc
```

### 3.3. Установка для ARM64

Для ARM64 заменить имя архива на:

```text
frp_0.71.0_linux_arm64.tar.gz
```

Его SHA256:

```text
f33c293c275d8fc68c654b6fba8f10b2551d6463d09a9fc9cffb7227eae82266
```

### 3.4. Проверка установки

```bash
frps --version
frpc --version
```

Оба бинарника должны вывести:

```text
0.71.0
```

### 3.5. Создать системного пользователя

Выполнить на обоих VPS:

```bash
sudo useradd \
  --system \
  --no-create-home \
  --shell /usr/sbin/nologin \
  frp

sudo install -d \
  -o root \
  -g frp \
  -m 0750 \
  /etc/frp
```

Если пользователь `frp` уже существует, повторно создавать его не нужно.

## 4. Создать общий токен

Выполнить один раз на любом VPS:

```bash
openssl rand -hex 32
```

Полученную строку сохранить и вставить в конфиги обоих VPS. Токен нельзя публиковать вместе с конфигурацией или логами.

Далее он обозначен как:

```text
FRP_SECRET_TOKEN
```

## 5. Настройка российского VPS: `frps`

Все команды этого раздела выполнять **на российском VPS `170.168.0.00`**.

### 5.1. Создать конфигурацию сервера

**На этом шаге выбери транспорт между VPS:** TCP (по умолчанию) или QUIC. Он не меняет AWG/UDP-порт `12345`: это отдельный порт, открываемый для Keenetic.

- **TCP:** оставь `quicBindPort` закомментированным. Клиент позже использует `transport.protocol = "tcp"`.
- **QUIC:** раскомментируй `quicBindPort = 7443`. Клиент позже использует `transport.protocol = "quic"`. При этом `bindPort = 7443` продолжит принимать TCP-соединения, поэтому можно оставить оба транспорта для отката. QUIC использует **UDP**, а TCP — **TCP**, конфликтов между ними нет.

```bash
sudo nano /etc/frp/frps.toml
```

Содержимое (вариант TCP по умолчанию; для QUIC убрать `# ` перед `quicBindPort`):

```toml
bindAddr = "0.0.0.0"
bindPort = 7443

# Для QUIC раскомментировать следующую строку:
# quicBindPort = 7443

proxyBindAddr = "0.0.0.0"
udpPacketSize = 1500

allowPorts = [
  { single = 12345 }
]

maxPortsPerClient = 2

auth.method = "token"
auth.token = "FRP_SECRET_TOKEN"

transport.tls.force = true

log.to = "console"
log.level = "info"
log.disablePrintColor = true
```

Выставить права:

```bash
sudo chown root:frp /etc/frp/frps.toml
sudo chmod 0640 /etc/frp/frps.toml
```

Проверить синтаксис:

```bash
sudo -u frp /usr/local/bin/frps verify -c /etc/frp/frps.toml
```

Ожидаемый результат:

```text
frps: the configuration file /etc/frp/frps.toml syntax is ok
```

### 5.2. Создать службу `frps`

```bash
sudo nano /etc/systemd/system/frps.service
```

Содержимое:

```ini
[Unit]
Description=FRP server for AWG relay
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=frp
Group=frp
ExecStart=/usr/local/bin/frps -c /etc/frp/frps.toml
Restart=always
RestartSec=5
NoNewPrivileges=true
PrivateTmp=true
ProtectHome=true

[Install]
WantedBy=multi-user.target
```

Загрузить и запустить службу:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now frps
sudo systemctl status frps --no-pager -l
```

Проверить порт транспорта FRP:

```bash
# TCP (слушается при наличии bindPort)
sudo ss -lntp | grep ':7443'

# QUIC (появится, если задан quicBindPort)
sudo ss -lunp | grep ':7443'
```

При TCP достаточно слушающего `7443/tcp`; при QUIC **обязательно** должен слушаться `7443/udp`. TCP-порт может слушаться и в варианте QUIC — это нормально. UDP-порт AWG-реле `12345` пока может отсутствовать: его откроет `frps` после успешного подключения иностранного `frpc`.

Логи:

```bash
sudo journalctl -u frps -n 50 --no-pager
sudo journalctl -u frps -f
```

### 5.3. Настроить и включить UFW

Все команды этого подраздела выполнять **на российском VPS**. Порты также должны быть разрешены в firewall/security group панели провайдера, если там настроена фильтрация.

| Протокол | Порт | Источник | Назначение |
|---|---:|---|---|
| TCP | `22` (или фактический SSH-порт) | любой адрес | SSH-администрирование VPS |
| TCP | `7443` | только иностранный VPS `179.255.000.000` | FRP по TCP; оставить также для отката с QUIC |
| UDP | `7443` | только иностранный VPS `179.255.000.000` | **только при выборе QUIC** |
| UDP | `12345` | домашний IP либо весь интернет | AWG от Keenetic (это **не** порт QUIC) |

**Важно:** сначала добавить правило для SSH, и только потом включать UFW. Текущую SSH-сессию не закрывать до проверки нового подключения.

#### 5.3.1. Проверить SSH-порт

```bash
sudo sshd -T | grep '^port '
```

Если результат — `port 22`, в командах ниже использовать `22`. Если порт другой — **заменить `22` на фактический порт**.

#### 5.3.2. Задать политики и разрешить нужные соединения

Правила `ufw allow` можно добавлять, даже если UFW ещё `inactive`.

```bash
# Запрещать неразрешённые входящие подключения
sudo ufw default deny incoming

# Разрешить исходящие подключения
sudo ufw default allow outgoing

# ВАЖНО: сначала разрешить SSH (при необходимости заменить 22)
sudo ufw allow 22/tcp

# FRP по TCP (оставить и при QUIC для возможности отката)
sudo ufw allow from 179.255.000.000 to any port 7443 proto tcp

# ТОЛЬКО ПРИ QUIC: открыть тот же номер порта, но по UDP
# Выполни эту строку, если включил quicBindPort в frps.toml:
# sudo ufw allow from 179.255.000.000 to any port 7443 proto udp

# AWG: UDP-порт для подключений Keenetic
sudo ufw allow 12345/udp
```

Здесь, как и в остальных разделах руководства, IP-адреса замаскированы: **перед выполнением команд нужно подставить реальные адреса**. Для QUIC нужно **выполнить** закомментированную в примере команду `ufw allow ... proto udp` (без `#`); одного разрешения `7443/tcp` недостаточно. Проверь также UDP `7443` в firewall панели провайдера, если он используется.

Если домашний адрес статический, можно **вместо** `sudo ufw allow 12345/udp` разрешить доступ только с него (в первоначальной диагностике он обозначен как `188.242.000.000`):

```bash
sudo ufw allow from 188.242.000.000 to any port 12345 proto udp
```

При динамическом домашнем IP лучше оставить общее разрешение `12345/udp`, иначе после смены адреса туннель перестанет работать. Если общее правило уже добавлено, новое ограниченное правило само по себе его не отменит — сначала нужно удалить общее правило командой `sudo ufw delete allow 12345/udp`.

#### 5.3.3. Проверить правила и включить UFW

**Перед включением убедиться, что правило для SSH-порта присутствует:**

```bash
sudo ufw show added
```

Если SSH-порт разрешён и список правильный:

```bash
sudo ufw enable
```

UFW может предупредить, что включение прервёт SSH-соединения. Подтвердить `y` **только после проверки правила SSH**.

Если UFW уже активен, повторно включать его не требуется: новые правила `ufw allow` применяются сразу.

Проверить состояние:

```bash
sudo ufw status verbose
```

Ожидается `Status: active`, политика `deny (incoming), allow (outgoing)` и разрешения примерно такого вида:

```text
7443/tcp    ALLOW IN    179.255.000.000
12345/udp   ALLOW IN    Anywhere
22/tcp      ALLOW IN    Anywhere
```

Если выбран QUIC, дополнительно должно быть правило `7443/udp ALLOW IN 179.255.000.000`. TCP-правило при этом можно не удалять.

При включённой поддержке IPv6 для общих правил могут появиться дополнительные строки `(v6)`. Это нормально.

#### 5.3.4. Проверить SSH из второго терминала

**Не закрывая текущую SSH-сессию**, открыть второй терминал и подключиться к российскому VPS:

```bash
ssh kasmas@170.168.0.00
```

Если SSH работает на нестандартном порту, добавить `-p ПОРТ`.

Если новое SSH-подключение не проходит, в ещё открытой сессии можно временно отключить UFW:

```bash
sudo ufw disable
```

После успешной проверки UFW будет автоматически загружать сохранённые правила при перезагрузке сервера. При добавлении второго UDP-реле (раздел 11) нужно также разрешить его порт в firewall.

## 6. Настройка иностранного VPS: `frpc`

Все команды этого раздела выполнять **на иностранном VPS `freedom`**.

### 6.1. Проверить доступность российского `frps`

**Для TCP** можно проверить подключение к `7443/tcp`:

```bash
nc -vz 170.168.0.00 7443
```

Ожидается сообщение об открытом порте:

```text
170.168.0.00 7443 open
```

**Для QUIC** такой `nc -vz` проверяет только TCP и не подтверждает доступность QUIC. На российском VPS проверь `sudo ss -lunp | grep ':7443'` и разрешение `7443/udp` в UFW и firewall провайдера. После запуска `frpc` проверь журнал клиента и двусторонние UDP-пакеты на `7443` (см. раздел 6.4).

В конкретном случае `nc` показывал имя `proxy.kosumosu.ru`. Это результат обратного DNS/PTR для IP и не означает, что такой домен настроен. В конфиге используется непосредственно IP `170.168.0.00`.

Если соединение не устанавливается, проверить:

- состояние `frps`;
- firewall ОС российского VPS;
- firewall/security group в панели российского провайдера;
- правильность IP и порта выбранного транспорта (TCP или UDP/QUIC).

### 6.2. Проверить локальный AWG-сервер

```bash
sudo docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
sudo docker port amnezia-awg2 12345/udp
```

Должна присутствовать публикация примерно такого вида:

```text
0.0.0.0:12345->12345/udp
```

### 6.3. Создать конфигурацию клиента

**Выбери тот же транспорт, который подготовлен на российском `frps` (раздел 5.1):**

- **TCP:** оставить активной строку `transport.protocol = "tcp"`.
- **QUIC:** закомментировать строку `"tcp"` и раскомментировать `transport.protocol = "quic"`. У российского `frps` должен быть включён `quicBindPort = 7443`, а UDP-порт `7443` открыт в firewall. В TOML должна остаться **ровно одна активная** строка `transport.protocol`.

```bash
sudo nano /etc/frp/frpc.toml
```

Содержимое (по умолчанию TCP):

```toml
user = "awg-ch"

serverAddr = "170.168.0.00"
serverPort = 7443

loginFailExit = false
udpPacketSize = 1500

auth.method = "token"
auth.token = "FRP_SECRET_TOKEN"

# Выбор транспорта: оставить ровно одну активную строку
transport.protocol = "tcp"
# transport.protocol = "quic"
transport.tls.enable = true

log.to = "console"
log.level = "info"
log.disablePrintColor = true

[[proxies]]
name = "awg-ch"
type = "udp"
localIP = "127.0.0.1"
localPort = 12345
remotePort = 12345
```

Где:

- `serverAddr` — российский VPS;
- `localIP` и `localPort` — AWG на иностранном VPS;
- `remotePort` — UDP-порт, который FRP откроет на российском VPS;
- `serverPort = 7443` — порт подключения `frpc → frps`: TCP при `transport.protocol = "tcp"` либо UDP/QUIC при `"quic"`;
- `transport.tls.enable = true` оставляем в примере без изменений; QUIC сам использует TLS как часть протокола.

Выставить права и проверить конфиг:

```bash
sudo chown root:frp /etc/frp/frpc.toml
sudo chmod 0640 /etc/frp/frpc.toml

sudo -u frp /usr/local/bin/frpc verify -c /etc/frp/frpc.toml
```

### 6.4. Создать службу `frpc`

```bash
sudo nano /etc/systemd/system/frpc.service
```

Содержимое:

```ini
[Unit]
Description=FRP client for AWG relay
After=network-online.target docker.service
Wants=network-online.target

[Service]
Type=simple
User=frp
Group=frp
ExecStart=/usr/local/bin/frpc -c /etc/frp/frpc.toml
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Запустить:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now frpc
sudo systemctl status frpc --no-pager -l
```

Посмотреть лог:

```bash
sudo journalctl -u frpc -n 50 --no-pager
```

В успешном случае в журнале будут сообщения приблизительно следующего содержания:

```text
login to server success
proxy added
start proxy success
```

**Если выбран QUIC**, на российском VPS проверь реальные UDP-пакеты между серверами (подставить фактический IP иностранного VPS):

```bash
sudo tcpdump -ni any 'udp port 7443 and host 179.255.000.000'
```

Двусторонний обмен на `7443/udp` вместе с успешным подключением `frpc` подтверждает использование QUIC. На `12345/udp` при этом по-прежнему работает отдельный AWG-прокси для Keenetic. Если QUIC не соединяется, временно верни в `frpc.toml` `transport.protocol = "tcp"` и перезапусти `frpc`; правило TCP в UFW для этого оставлено.

## 7. Проверить регистрацию UDP-прокси

Выполнять **на российском VPS** после запуска `frpc`:

```bash
sudo ss -lunp | grep ':12345'
```

Теперь должен появиться UDP-порт `12345`, обслуживаемый `frps`.

Если порт не появился:

```bash
# На российском VPS
sudo journalctl -u frps -n 50 --no-pager

# На иностранном VPS
sudo journalctl -u frpc -n 50 --no-pager
```

Наиболее частые причины:

- токены в `frps.toml` и `frpc.toml` отличаются;
- порт `12345` отсутствует в `allowPorts`;
- `frpc` не может подключиться к `170.168.0.00:7443` по выбранному транспорту (для QUIC нужен открытый UDP `7443`);
- ошибка синтаксиса TOML;
- на российском VPS этот UDP-порт уже занят другим процессом.

## 8. Изменить AWG на Keenetic

FRP на Keenetic не устанавливается.

В конфигурации нужного AWG-туннеля изменить только строку `Endpoint`.

Было:

```ini
Endpoint = 179.255.000.000:12345
```

Стало:

```ini
Endpoint = 170.168.0.00:12345
```

Не менять:

- `PrivateKey`;
- `PublicKey`;
- `PresharedKey`;
- адреса интерфейса;
- `H1–H4`;
- `S1–S4`;
- `I1–I5`;
- `HeaderProtectionKey`;
- `PersistentKeepalive`;
- серверный `ListenPort`.

На иностранном AWG-сервере также ничего менять не требуется.

После сохранения включить или перезапустить туннель в AWG Manager.

## 9. Проверить работу всего маршрута

### 9.1. На российском VPS

```bash
sudo tcpdump -ni any 'udp port 12345'
```

Должны появляться пакеты в обе стороны:

```text
домашний_IP:случайный_порт > 170.168.0.00:12345
170.168.0.00:12345 > домашний_IP:случайный_порт
```

### 9.2. На иностранном VPS

```bash
sudo tcpdump -ni any 'udp port 12345'
```

А также:

```bash
sudo docker exec amnezia-awg2 awg show
```

Признаки успешной работы:

- появился `latest handshake`;
- увеличиваются `received` и `sent`;
- Keenetic получает доступ через туннель.

Адрес peer на AWG-сервере теперь может отображаться как `127.0.0.1:порт` или `172.x.x.x:порт`. Это нормально: непосредственным источником пакетов для AWG теперь является локальный `frpc` или Docker bridge.

### 9.3. На Keenetic

```sh
awg show opkgtun11
```

Проверить:

- наличие свежего `latest handshake`;
- рост обоих счётчиков трафика;
- отсутствие только исходящих байтов при нулевых входящих.

## 10. Если FRP подключён, но AWG-handshake отсутствует

На иностранном VPS:

```bash
sudo tcpdump -ni lo 'udp port 12345'
sudo tcpdump -ni any 'udp port 12345'
```

Если пакеты от `frpc` приходят, но не достигают контейнера через `127.0.0.1`, узнать адрес контейнера:

```bash
sudo docker inspect -f \
  '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' \
  amnezia-awg2
```

После этого временно заменить в `/etc/frp/frpc.toml`:

```toml
localIP = "127.0.0.1"
```

на полученный адрес, например:

```toml
localIP = "172.18.0.2"
```

Затем:

```bash
sudo -u frp /usr/local/bin/frpc verify -c /etc/frp/frpc.toml
sudo systemctl restart frpc
sudo journalctl -u frpc -n 50 --no-pager
```

Минус обращения прямо к адресу контейнера: он может измениться после пересоздания Docker-сети. Поэтому рабочий `127.0.0.1` предпочтительнее.

## 11. Добавить второй иностранный AWG-сервер

Отдельный российский VPS для второго AWG не требуется. Один `frps` может открыть несколько UDP-портов.

### 11.1. Российский VPS

В `/etc/frp/frps.toml` разрешить второй порт:

```toml
allowPorts = [
  { single = 12345 },
  { single = 34433 }
]
```

Проверить и перезапустить:

```bash
sudo -u frp /usr/local/bin/frps verify -c /etc/frp/frps.toml
sudo systemctl restart frps
sudo systemctl status frps --no-pager -l
```

В firewall российского VPS также разрешить `34433/udp`. Выбор транспорта самого `frpc → frps` для второго VPS независим: TCP использует `7443/tcp`, QUIC — `7443/udp` (см. разделы 5.1, 5.3 и 6.3). Не путать `34433/udp` (AWG от Keenetic) с `7443/udp` (QUIC между VPS).

### 11.2. Второй иностранный VPS

Установить `frpc` по тому же руководству и задать уникальные имя и внешний порт:

```toml
user = "awg-second"

serverAddr = "170.168.0.00"
serverPort = 7443

loginFailExit = false
udpPacketSize = 1500

auth.method = "token"
auth.token = "FRP_SECRET_TOKEN"

# Для QUIC вместо "tcp" указать "quic";
# на российском frps должен быть включён quicBindPort = 7443
transport.protocol = "tcp"
transport.tls.enable = true

[[proxies]]
name = "awg-second"
type = "udp"
localIP = "127.0.0.1"
localPort = ПОРТ_AWG_НА_ВТОРОМ_VPS
remotePort = 34433
```

Второй туннель на Keenetic должен использовать:

```ini
Endpoint = 170.168.0.00:34433
```

Итог:

```text
170.168.0.00:12345/udp → первый иностранный AWG
170.168.0.00:34433/udp → второй иностранный AWG
```

## 12. Повседневное обслуживание

### Статус служб

```bash
# Российский VPS
sudo systemctl status frps --no-pager -l

# Иностранный VPS
sudo systemctl status frpc --no-pager -l
```

### Последние логи

```bash
# Российский VPS
sudo journalctl -u frps -n 100 --no-pager

# Иностранный VPS
sudo journalctl -u frpc -n 100 --no-pager
```

### Перезапуск

```bash
# Российский VPS
sudo systemctl restart frps

# Иностранный VPS
sudo systemctl restart frpc
```

### Проверка портов

```bash
# Российский VPS: TCP-транспорт FRP
sudo ss -lntp | grep ':7443'

# Российский VPS: QUIC-транспорт FRP (если включён)
sudo ss -lunp | grep ':7443'

# Российский VPS: UDP-реле, появляется после регистрации frpc
sudo ss -lunp | grep ':12345'

# Иностранный VPS: публикация AWG из Docker
sudo docker port amnezia-awg2 12345/udp
```

## 13. Что важно помнить

1. Сначала запускается `frps` на российском VPS, затем `frpc` на иностранном. При выборе QUIC у `frps` нужно включить `quicBindPort`, у `frpc` выбрать `transport.protocol = "quic"`, а в firewall разрешить `7443/udp` только от иностранного VPS.
2. UDP-порт реле на российском VPS регистрируется клиентом и может исчезнуть, когда `frpc` отключён.
3. Токен в двух конфигурациях должен совпадать.
4. `udpPacketSize` на обеих сторонах должен совпадать.
5. На Keenetic меняется только `Endpoint`.
6. При добавлении второго иностранного сервера нужен другой `remotePort`.
7. Весь AWG-трафик теперь идёт дополнительным маршрутом через российский VPS, поэтому задержка может увеличиться.
8. При **TCP** FRP переносит AWG/UDP внутри TCP/TLS; потери пакетов могут приводить к дополнительным задержкам из-за повторной передачи TCP. При **QUIC** транспорт между VPS использует UDP и избегает части проблем TCP, но не гарантирует отсутствие лагов: надёжная доставка и повторные передачи существуют и в QUIC. Проверять фактическую стабильность обоих вариантов нужно на своём маршруте.

## 14. Замечание о TLS

В базовой конфигурации при TCP FRP шифрует соединение между `frpc` и `frps`, а сервер принудительно требует TLS. QUIC также использует TLS как встроенную часть протокола. Без отдельно настроенного CA клиент по умолчанию не проверяет личность `frps` по сертификату.

Для этой схемы риски дополнительно уменьшаются следующим:

- TCP `7443` (а при QUIC и UDP `7443`) разрешён только с IP иностранного VPS;
- используется длинный случайный токен;
- полезная нагрузка дополнительно зашифрована самим AWG.

При необходимости конфигурацию можно позднее усилить собственным CA и проверкой сертификата `frps`.

## 15. Официальные источники

- FRP: <https://github.com/fatedier/frp>
- Релизы: <https://github.com/fatedier/frp/releases>
- Настройка UDP-прокси: <https://gofrp.org/en/docs/features/tcp-udp/>
- Token-аутентификация: <https://gofrp.org/en/docs/features/common/authentication/>
- TLS: <https://gofrp.org/en/docs/features/common/network/network-tls/>
- TCP и QUIC как транспорт FRP: <https://gofrp.org/en/docs/features/common/network/network/>
- Использование systemd: <https://gofrp.org/en/docs/setup/systemd/>
