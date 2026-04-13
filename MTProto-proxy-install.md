# Установка MTProxy Proxy с FakeTLS без использования docker 

# Скачивание и сборка клиента

Здесь предполагается, что всё происходит из-под рута. Если делается из под юзера, то в некоторых командах нужно будет добавлять `sudo`

## Зависимости

    apt update 
    apt install -y git curl build-essential libssl-dev zlib1g-dev ca-certificates 

## Установка клиента

    cd /opt 
    git clone https://github.com/TelegramMessenger/MTProxy.git mtproxy 
    cd /opt/mtproxy 

## Сборка

    make 

## Получение конфигов

    mkdir -p /opt/mtproxy/data 
    curl -fsSL https://core.telegram.org/getProxySecret -o /opt/mtproxy/data/proxy-secret 
    curl -fsSL https://core.telegram.org/getProxyConfig -o /opt/mtproxy/data/proxy-multi.conf

## Питон скрипт для получения секретов: 

    nano gen_keys.py

```python
import secrets


def gen_secret():
   site = input('Type host: ').strip()
   encoded_host_hex = site.encode("ascii").hex()

   raw_secret = secrets.token_hex(16)
   user_secret = 'ee' + raw_secret + encoded_host_hex

   print(f'Secret for start MTProto Proxy: {raw_secret}')
   print(f'User secret: {user_secret}')


if __name__ == '__main__':
   gen_secret()
```

### запуск: 

    python3 gen_keys.py

вписать домен под который маскироваться, нажать `Enter`

## Секрет без питона

### Сырой секрет

    SECRET=$(openssl rand -hex 16) 
    echo "$SECRET"

**либо так:** 

    head -c 16 /dev/urandom | xxd -ps

### hex строка в ahcii домена

**через xxd:**

    site='ya.ru'
    printf '%s' "$site" | xxd -p -c 256

**либо od:**

    site='ya.ru'
    printf '%s' "$site" | od -An -tx1 | tr -d ' \n'

**либо hexdump:**

    site='ya.ru'
    printf '%s' "$site" | hexdump -ve '1/1 "%.2x"'

сырой секрет нужно будет указать в сервис-файле, а юзеру в ссылке отдавать такой: 

    ee + сырой_секрет + hex_строка_домена

## systemd сервис

дальше нужно создать systemd сервис файл: 

    touch /etc/systemd/system/mtproxy.service

содержимое: 

    [Unit]
    Description=MTProxy
    After=network.target
    Wants=network-online.target
    [Service]
    Type=simple
    WorkingDirectory=/opt/mtproxy/objs/bin
    ExecStart=/opt/mtproxy/objs/bin/mtproto-proxy -u nobody -p 8888 -H 443 --http-stats -S СЫРОЙ_СЕКРЕТ --domain ДОМЕН_КОТОРЫЙ_УКАЗАЛИ --aes-pwd /opt/mtproxy/data/proxy-secret /opt/mtproxy/data/proxy-multi.conf -M 1
    Restart=on-failure
    RestartSec=3
    LimitNOFILE=65535
    [Install]
    WantedBy=multi-user.target

* `-M 1` -- это один воркер
* `8888` -- порт для доступа к статистике `wget http://127.0.0.1:8888/stats -O FILE_NAME -q`

## запуск:

    systemctl daemon-reload 
    systemctl enable --now mtproxy 

## проверить как поднялось: 

    systemctl status mtproxy

или

    service mtproxy status

## ссылка для юзера: 

`tg://proxy?server=IP_ADDRESS&port=443&secret=USER_SECRET_STARTS_FROM_EE`

или

`https://t.me/proxy?server=IP_ADDRESS&port=443&secret=USER_SECRET_STARTS_FROM_EE`

## Обновление конфигов

телеграм рекомендует обновлять конфигурацию раз в сутки: 

    tee /etc/cron.d/mtproxy-update >/dev/null <<EOF 
    17 4 * * * root curl -fsSL https://core.telegram.org/getProxyConfig -o /opt/mtproxy/data/proxy-multi.conf && systemctl restart mtproxy 
    EOF

## статистика через телегу

Если нужен не только личный доступ, но и статистика через Telegram, тогда можно зарегистрировать прокси у `@MTProxyBot` и добавить тег через параметр `-P`. Для личного сервера без монетизации шаг необязателен.

# Проблемы

## Падает сервис с ошибкой по assertion PID в логах

Если сервис упал с ошибкой в логах с такой строкой: 

    mtproto-proxy: common/pid.c:42: init_common_PID: Assertion '!(p & 0xffff0000)' failed.

то проблема в том, что в исходниках `MTProxy` в `init_common_PID()` стоит `assert (!(p & 0xffff0000));` сразу после `getpid()`, то есть процесс аварийно завершается, если `PID` больше `65535`. Это известная проблема MTProxy.

### Проверка: 

    cat /proc/sys/kernel/pid_max

если значение больше `65535`

    sysctl -w kernel.pid_max=65535

сбрасываем счётчик падений и перезапускаем сервис

    systemctl reset-failed mtproto-proxy.service
    systemctl restart mtproto-proxy.service
    systemctl status mtproto-proxy.service -l --no-pager

### Решение:

Если это помогло, то прописываем в конфиг ограничнение:

    printf 'kernel.pid_max = 65535\n' > /etc/sysctl.d/99-mtproxy.conf
    sysctl --system

возможно надо рестартануть: 

    systemctl restart mtproto-proxy.service
