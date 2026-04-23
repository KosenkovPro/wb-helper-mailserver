# wb-helper-mailserver

Почтовый сервер для домена `wb-helper.tech` на базе [docker-mailserver](https://github.com/docker-mailserver/docker-mailserver) (postfix + dovecot + rspamd/spamassassin + opendkim + fail2ban).

Разворачивается на сервере `lion` (192.168.1.4) через GitHub Actions. В отличие от `wb-helper-infra`, публичный доступ — **без** Apache reverse proxy: SMTP/IMAP — это не HTTP, порты 25/465/587/993 проброшены наружу напрямую.

## Состав

| Порт | Протокол | Назначение | Наружу |
|---|---|---|---|
| `25` | SMTP | Приём почты от других MTA | да |
| `465` | SMTPS (implicit TLS) | Отправка от клиентов | да |
| `587` | SMTP Submission (STARTTLS) | Отправка от клиентов | да |
| `993` | IMAPS | Чтение ящиков клиентами | да |

POP3 и незашифрованный IMAP (143) отключены в `mailserver.env`.

## Предусловия перед деплоем

### 1. DNS (на DNS-провайдере домена)

| Тип | Имя | Значение |
|---|---|---|
| `A` | `mail.wb-helper.tech` | публичный IP сервера lion |
| `MX` | `wb-helper.tech` | `10 mail.wb-helper.tech` |
| `TXT` | `wb-helper.tech` | `v=spf1 mx -all` |
| `TXT` | `_dmarc.wb-helper.tech` | `v=DMARC1; p=quarantine; rua=mailto:postmaster@wb-helper.tech` |
| `TXT` | `mail._domainkey.wb-helper.tech` | DKIM-ключ (сгенерируется на шаге «первичная настройка») |
| `PTR` | (reverse) | `mail.wb-helper.tech` (настраивается у хостера!) |

PTR **обязателен** — без него Gmail/Yandex будут бить письма в спам или отклонять.

### 2. Firewall / port-forwarding на роутере

Прокинуть на lion (192.168.1.4) TCP: `25`, `465`, `587`, `993`. Если используется отдельный публичный IP — укажи его в `MAIL_BIND_IP`.

### 3. TLS-сертификат

На lion должен стоять `certbot` и быть выпущен сертификат:

```bash
sudo apt install certbot
sudo certbot certonly --standalone -d mail.wb-helper.tech
# cert: /etc/letsencrypt/live/mail.wb-helper.tech/{fullchain,privkey}.pem
```

docker-mailserver подхватывает их автоматически при `SSL_TYPE=letsencrypt`. Автообновление certbot → `systemctl enable --now certbot.timer`.

## Локальный запуск

```bash
cp .env.example .env   # заполнить MAIL_HOSTNAME, POSTMASTER_ADDRESS
# Для локалки проще SSL_TYPE=self-signed или выключить TLS в mailserver.env
docker compose up -d
docker compose logs -f mailserver
```

## Первичная настройка (на сервере, после первого деплоя)

```bash
cd /opt/wb-helper-mailserver

# 1. Создать ящик (пароль интерактивно)
docker exec -it wb-mailserver setup email add postmaster@wb-helper.tech

# 2. Сгенерировать DKIM-ключ
docker exec -it wb-mailserver setup config dkim

# 3. Опубликовать DKIM-запись в DNS.
cat config/opendkim/keys/wb-helper.tech/mail.txt
# → скопировать TXT-значение в DNS как mail._domainkey.wb-helper.tech

# 4. Проверить работу
docker exec -it wb-mailserver setup debug fetchmail
docker exec -it wb-mailserver postconf -n | head
```

Созданные `config/postfix-accounts.cf` и `config/opendkim/…` нужно **закоммитить в git** (пароли там хранятся в виде SHA512-хешей, DKIM private key — помечен `.gitignore` если не хотим его в git; по умолчанию docker-mailserver кладёт приватник рядом — для прод-безопасности его лучше не пушить).

> **Важно.** Приватные DKIM-ключи (`config/opendkim/keys/**/*.private`) коммитить в публичный репозиторий не стоит. Либо держи репо приватным, либо добавь их в `.gitignore` и переноси отдельно (через secret-хранилище).

## Деплой

`.github/workflows/deploy.yml` триггерится по `push` в `main` с изменениями в `docker-compose.yml`, `mailserver.env`, `config/**`:

1. SSH на `lion` под деплой-пользователем.
2. `git reset --hard origin/main` в `${MAIL_DEPLOY_PATH}` (обычно `/opt/wb-helper-mailserver`).
3. Запись `.env` из GitHub Secrets.
4. `docker compose pull && up -d --remove-orphans`.
5. Ждём статус контейнера `healthy` (до 3 минут — spamassassin/amavis долго прогреваются).

### GitHub Secrets

| Имя | Значение | Пример |
|---|---|---|
| `SSH_HOST` | Внешний IP сервера | `95.165.9.178` |
| `SSH_PORT` | Проброс до `lion:22` | `49154` |
| `SSH_USER` | Деплой-пользователь | `lion` |
| `SSH_PRIVATE_KEY` | Приватный ключ (deploy key на lion) | `-----BEGIN ...` |
| `MAIL_DEPLOY_PATH` | Путь на lion | `/opt/wb-helper-mailserver` |
| `MAIL_HOSTNAME` | FQDN MX | `mail.wb-helper.tech` |
| `POSTMASTER_ADDRESS` | Постмастер | `postmaster@wb-helper.tech` |
| `MAIL_BIND_IP` | IP для биндинга портов | `0.0.0.0` |
| `SSL_TYPE` | Тип TLS | `letsencrypt` |

### Первичная настройка сервера lion

```bash
# 1. Склонировать репо
sudo mkdir -p /opt/wb-helper-mailserver
sudo chown lion:lion /opt/wb-helper-mailserver
cd /opt/wb-helper-mailserver
git clone git@github.com:<org>/wb-helper-mailserver.git .

# 2. Certbot (см. выше)

# 3. Первый пуш в main триггернёт деплой.
```

## Бэкап

Ежедневный tar-бэкап `./docker-data/` (письма + состояние) на внешнее хранилище — настраивается отдельно (см. `wb-helper-infra/README.md` для общего подхода к бэкапам).

## Проверка, что сервер «виден» миру

```bash
# MX/DNS
dig +short MX wb-helper.tech
dig +short mail.wb-helper.tech

# SMTP-баннер с внешней машины
nc -v mail.wb-helper.tech 25
# → 220 mail.wb-helper.tech ESMTP

# Репутация и настройки (spf/dkim/dmarc)
# https://www.mail-tester.com/ — отправить тестовое письмо, посмотреть скор.
```
