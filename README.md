# wb-helper-mailserver

Почтовый сервер для домена `wb-helper.tech` на базе [docker-mailserver](https://github.com/docker-mailserver/docker-mailserver) (postfix + dovecot + rspamd/spamassassin + opendkim + fail2ban).

Разворачивается **на сервере БД `bison`** (внешний доступ `95.165.9.178:49153`) через **GitHub Actions**. Публичный доступ — без Apache reverse proxy `bouncer77` (он только для HTTP): SMTP/IMAP — это не HTTP, порты 25/465/587/993 проброшены с роутера напрямую на bison.

### Карта инфраструктуры WB

| Роль | Внутр. имя | Внешний доступ | Что крутится |
|---|---|---|---|
| Сервер приложений | `lion` (192.168.1.4) | `95.165.9.178:49154` | wb-backend, wb-helper-infra (Grafana/Prometheus/Loki) |
| **Сервер БД** | `bison` | `95.165.9.178:49153` (user `bison`) | Postgres + **wb-helper-mailserver** |
| Apache reverse proxy | `bouncer77` (192.168.1.7) | `:9811` (LAN) | `*.wb-helper.tech` HTTPS — почту **не** проксирует |

## Состав

| Порт | Протокол | Назначение | Наружу |
|---|---|---|---|
| `25` | SMTP | Приём почты от других MTA | да |
| `465` | SMTPS (implicit TLS) | Отправка от клиентов | да |
| `587` | SMTP Submission (STARTTLS) | Отправка от клиентов | да |
| `993` | IMAPS | Чтение ящиков клиентами | да |

POP3 и незашифрованный IMAP (143) отключены в `mailserver.env`.

---

## 1. DNS — что указать у регистратора домена

> Замени `wb-helper.tech` на свой домен и `<PUBLIC_IP>` на публичный IP сервера БД (или IP, на который мэйл проброшен снаружи).

| # | Тип | Имя (host) | Значение | TTL | Зачем |
|---|---|---|---|---|---|
| 1 | `A` | `mail` | `<PUBLIC_IP>` | 3600 | A-запись хоста MX |
| 2 | `MX` | `@` | `mail.wb-helper.tech.` приоритет `10` | 3600 | Куда другие MTA шлют почту для домена |
| 3 | `TXT` | `@` | `v=spf1 mx -all` | 3600 | SPF — разрешает отправку только из MX-хоста |
| 4 | `TXT` | `_dmarc` | см. ниже (раздел про «недопустимые символы») | 3600 | DMARC — политика для писем без SPF/DKIM |
| 5 | `TXT` | `mail._domainkey` | (значение появится после **первого деплоя**, шаг 4) | 3600 | DKIM — подпись писем приватным ключом |

### PTR (reverse DNS) — у **хостера/провайдера IP**, не у регистратора

| Тип | Имя | Значение |
|---|---|---|
| `PTR` | reverse `<PUBLIC_IP>` (`in-addr.arpa`) | `mail.wb-helper.tech.` |

**PTR обязателен.** Без него Gmail/Yandex/Mail.ru отбрасывают письма с `550 5.7.1`. Настраивается в личном кабинете VPS.

### 1.1. Ошибка «текст содержит недопустимые символы» при добавлении `_dmarc`

Это особенность UI многих регистраторов (REG.RU, RU-CENTER, Beget, GoDaddy и др.) — они валидируют поле «значение TXT» и спотыкаются о `;`, `:`, `@` или пробелы. Сама запись DMARC **обязана** содержать `;` (RFC 7489), поэтому правильное решение — обернуть значение в **двойные кавычки**, тогда UI пропускает строку как единый литерал.

**Пробуй варианты в таком порядке** — какой проглотит, тот и используй (DNS-сервер всё равно отдаёт байты как есть):

```
# 1. Минимальный DMARC в кавычках — рекомендую начать с него:
"v=DMARC1; p=none; rua=mailto:postmaster@wb-helper.tech"

# 2. Без кавычек, минимальный (если UI кавычки не любит):
v=DMARC1; p=none; rua=mailto:postmaster@wb-helper.tech

# 3. Если ругается на @ в rua — на старте можно вообще без rua:
v=DMARC1; p=none

# 4. Если ругается на ; — у некоторых российских регистраторов есть отдельная
#    форма "DMARC-запись" в админке (REG.RU: «DNS → Запись → DMARC»),
#    где поля заполняются по одному и ; ставится автоматически.
```

Почему `p=none` на старте, а не `p=quarantine`: первые 1–2 недели смотришь rua-отчёты, убеждаешься, что собственная почта проходит, и только потом ужесточаешь до `quarantine`, потом `reject`. Иначе свои же письма уйдут в спам у получателей.

**Полезно:** валидаторы DMARC — https://mxtoolbox.com/dmarc.aspx или https://dmarcian.com/dmarc-inspector/ — покажут, что DNS реально вернул, после того как запись применится (TTL).

### 1.2. Проверка после применения

```bash
dig +short MX wb-helper.tech                  # → 10 mail.wb-helper.tech.
dig +short A mail.wb-helper.tech              # → <PUBLIC_IP>
dig +short TXT wb-helper.tech                 # → "v=spf1 mx -all"
dig +short TXT _dmarc.wb-helper.tech          # → "v=DMARC1; p=none; ..."
dig +short -x <PUBLIC_IP>                     # → mail.wb-helper.tech. (PTR)
```

Финальная проверка репутации — отправить письмо на https://www.mail-tester.com/, цель — **10/10**.

---

## 2. GitHub Actions — что указать в проекте

Пайплайн `.github/workflows/deploy.yml` уже в репо. Триггерится по `push` в `main` с изменениями в `docker-compose.yml`, `mailserver.env`, `config/**` (или вручную из вкладки **Actions** через `workflow_dispatch`).

### Secrets (Settings → Secrets and variables → Actions → New repository secret)

| Имя | Значение / пример |
|---|---|
| `SSH_HOST` | `95.165.9.178` |
| `SSH_PORT` | `49153` (NAT-проброс на bison:22) |
| `SSH_USER` | `bison` (или отдельный `deploy`-пользователь на bison — см. ниже) |
| `SSH_PRIVATE_KEY` | приватный ключ ed25519 в PEM, **с переводами строк** |
| `MAIL_DEPLOY_PATH` | `/opt/wb-helper-mailserver` |
| `MAIL_HOSTNAME` | `mail.wb-helper.tech` |
| `POSTMASTER_ADDRESS` | `postmaster@wb-helper.tech` |
| `MAIL_BIND_IP` | `0.0.0.0` (или конкретный публичный IP, если у bison несколько интерфейсов) |
| `SSL_TYPE` | `letsencrypt` |

> Для `SSH_PORT` **49154 — это lion** (сервер приложений). Не путать. Для mailserver-деплоя — **49153** (bison).

> GitHub Secrets корректно сохраняют многострочные значения — приватный ключ вставляй целиком, включая `-----BEGIN OPENSSH PRIVATE KEY-----` и `-----END OPENSSH PRIVATE KEY-----`.

### Подготовка сервера bison один раз

Все команды выполняются с админ-машины. На bison ходим через `ssh -p 49153 bison@95.165.9.178` (пароль есть у владельца, в репо не коммитим).

```bash
# === На локальной машине: сгенерировать deploy-ключ для CI ===
ssh-keygen -t ed25519 -f ~/.ssh/wb-mailserver-deploy -N "" \
  -C "github-actions@wb-helper-mailserver"

# === На bison (sshd слушает 22 внутри, наружу проброшен 49153) ===
ssh -p 49153 bison@95.165.9.178

# 1. Деплой-пользователь и каталог
sudo useradd -m -s /bin/bash deploy
sudo usermod -aG docker deploy
sudo mkdir -p /opt/wb-helper-mailserver
sudo chown deploy:deploy /opt/wb-helper-mailserver

# 2. Положить публичный ключ
sudo -u deploy mkdir -p /home/deploy/.ssh
sudo -u deploy chmod 700 /home/deploy/.ssh
# Вставить содержимое ~/.ssh/wb-mailserver-deploy.pub (с локальной машины):
sudo -u deploy tee /home/deploy/.ssh/authorized_keys >/dev/null
sudo -u deploy chmod 600 /home/deploy/.ssh/authorized_keys

# 3. Клон репо в /opt (deploy-key/пуллер на github-стороне или https с PAT)
sudo -u deploy git clone https://github.com/<org>/wb-helper-mailserver.git \
  /opt/wb-helper-mailserver

# 4. Certbot для TLS (см. раздел 3)
sudo apt install -y certbot
sudo certbot certonly --standalone -d mail.wb-helper.tech
sudo systemctl enable --now certbot.timer

# 5. На роутере: проброс TCP 25/465/587/993 на bison (не на bouncer77 и не на lion).
exit
```

Содержимое `~/.ssh/wb-mailserver-deploy` (приватный ключ целиком, включая `-----BEGIN/END-----`) → в GitHub Secret `SSH_PRIVATE_KEY`. `SSH_USER=deploy`.

После этого первый push в `main` (или ручной `workflow_dispatch`) развернёт сервис.

---

## 3. TLS-сертификат

`SSL_TYPE=letsencrypt` — docker-mailserver сам подхватывает `/etc/letsencrypt/live/${MAIL_HOSTNAME}/{fullchain,privkey}.pem`. Сертификат выпускает **certbot на хосте**, не контейнер. Автообновление через `certbot.timer`.

> На время выпуска cert порт `80` должен быть свободен (или используй DNS-01 challenge).

---

## 4. Первичная настройка после первого успешного деплоя

```bash
ssh -p 49153 deploy@95.165.9.178      # bison
cd /opt/wb-helper-mailserver

# 1. Создать ящик postmaster
docker exec -it wb-mailserver setup email add postmaster@wb-helper.tech
# → введи пароль; хеш запишется в config/postfix-accounts.cf

# 2. Сгенерировать DKIM-ключ
docker exec -it wb-mailserver setup config dkim

# 3. Получить значение DKIM TXT для DNS
cat config/opendkim/keys/wb-helper.tech/mail.txt
# → скопировать всё, что в скобках, в одну строку, и добавить как
#   TXT mail._domainkey.wb-helper.tech = "v=DKIM1; k=rsa; p=MIIBIjAN..."

# 4. Закоммитить публичные конфиги (postfix-accounts.cf — хеши, не plaintext)
cd /opt/wb-helper-mailserver
git add config/postfix-accounts.cf
git commit -m "ops: add postmaster account"
git push origin main
```

> **Безопасность.** Приватные DKIM-ключи (`config/opendkim/keys/**/*.private`) в публичный git **не пуши**. Если репо приватный — допустимо; иначе добавь их в `.gitignore` и держи только на сервере.

---

## 5. Локальный запуск

```bash
cp .env.example .env   # заполнить MAIL_HOSTNAME, POSTMASTER_ADDRESS
# Для локалки проще SSL_TYPE=self-signed
docker compose up -d
docker compose logs -f mailserver
```

---

## 6. Проверка работоспособности

```bash
# 1. Контейнер healthy?
ssh -p 49153 deploy@95.165.9.178 'docker inspect -f "{{.State.Health.Status}}" wb-mailserver'

# 2. SMTP-баннер с внешней машины
nc -v mail.wb-helper.tech 25
# → 220 mail.wb-helper.tech ESMTP Postfix

# 3. Полный тест репутации
# Отправить письмо на адрес с https://www.mail-tester.com → ожидаемый скор 10/10.

# 4. Логи
ssh -p 49153 deploy@95.165.9.178 'cd /opt/wb-helper-mailserver && docker compose logs --tail=200 mailserver'
```

---

## 7. Бэкап

`./docker-data/` (письма + состояние + конфиг) — критичная директория. Настрой ежедневный tar-снимок на внешнее хранилище. DKIM-ключи и `postfix-accounts.cf` — в первую очередь.
