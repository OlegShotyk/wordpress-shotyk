# Звіт з розрахунково-графічної роботи

**Дисципліна:** Операційні системи
**Заклад:** НУ «Львівська політехніка», Інститут комп'ютерних наук та інформаційних технологій, кафедра САП
**Виконав:** Олег Шотик
**Рік:** 2026

**Тема:** Розгортання WordPress-вебаплікації з використанням Docker Compose.

**Мета:** дослідити та реалізувати встановлення WordPress за допомогою Docker Compose; описати архітектуру контейнерів (база даних, PHP-FPM, Nginx, SSL); проаналізувати `docker-compose.yml`, `.env` і конфігурацію Nginx; показати запуск, налаштування SSL та перевірку роботи на двох операційних системах.

---

## 1. Середовище виконання

| Роль | ОС | Docker | Мережа |
|---|---|---|---|
| Linux-машина | Ubuntu 26.04.1 LTS, VM VirtualBox `UbuntuVM` (hostname `ubuntu-shotyk`) | Docker Engine 29.9.3, Docker Compose v5.7.1 (плагін) | NAT; `office` 192.168.1.20/24; host-only 192.168.56.20/24 |
| Windows-машина | Windows 10 (фізичний хост) | Docker Desktop 29.9.3 (бекенд WSL2, ядро 6.18.58), Compose v5.7.0 | host-only 192.168.56.1/24 |
| Спроба Windows у VM | Windows 10 Pro, VM VirtualBox `Win10Client` | Docker Desktop встановлено, але двигун не запустився (див. розд. 7) | `office` 192.168.1.2/24 |

Скрізь використано однакові назви: домен `oleg-shotyk.local`, каталог проєкту `wordpress-shotyk`, сервіси й контейнери `db-shotyk`, `wordpress-shotyk`, `webserver-shotyk`.

## 2. Архітектура

```
 браузер ──HTTP:80──►┐
                     │  webserver-shotyk (nginx:1.15.12-alpine)
 браузер ─HTTPS:443─►┘   ├─ 80  → 301 redirect на https
                         ├─ 443 → TLS (oleg-shotyk.crt / .key з ./nginx-conf)
                         └─ *.php → FastCGI wordpress-shotyk:9000
                                         │
                         wordpress-shotyk (wordpress:5.1.1-fpm-alpine, PHP-FPM)
                                         │ mysqli → db-shotyk:3306
                         db-shotyk (mysql:8.0, mysql_native_password)

 Томи:   wordpress → /var/www/html (спільний для nginx і php-fpm)
         dbdata    → /var/lib/mysql
 Мережа: app-network (bridge), сервіси звертаються один до одного за іменами
```

## 3. Конфігураційні файли

- [`nginx-conf/nginx.conf`](nginx-conf/nginx.conf) має два блоки `server`. Перший слухає порт 80 і повертає `301` на `https://$host$request_uri`. Другий слухає 443 з `ssl_certificate`/`ssl_certificate_key` з `/etc/nginx/conf.d/`, має `root /var/www/html`, `try_files ... /index.php$is_args$args` для «чистих» URL і `fastcgi_pass wordpress-shotyk:9000` для PHP. Він також забороняє `.ht*`, вимикає журнали для `favicon.ico`/`robots.txt` і задає `expires max` для статичних файлів.
- [`docker-compose.yml`](docker-compose.yml) описує три сервіси з `restart: unless-stopped`, `env_file: .env`, іменовані томи `wordpress`/`dbdata` і мережу `app-network`. MySQL запускається з `--default-authentication-plugin=mysql_native_password`, бо PHP-клієнт у WordPress 5.1.1 не підтримує `caching_sha2_password`.
- `.env` містить `MYSQL_DATABASE`, `MYSQL_ROOT_PASSWORD`, `MYSQL_USER`, `MYSQL_PASSWORD`. Шаблон без паролів лежить у [`.env.example`](.env.example).
- [`.gitignore`](.gitignore) виключає `.env`, приватний ключ `nginx-conf/*.key` і локальний файл з обліковими даними. [`.dockerignore`](.dockerignore) містить `.env`, `.git`, `docker-compose.yml`, `.dockerignore`.

## 4. Хід роботи на Ubuntu 26.04 (VM)

| # | Крок | Знімок |
|---|---|---|
| 1 | Версії ОС і Docker, IP-адреси, запис `127.0.0.1 oleg-shotyk.local` у `/etc/hosts` | ![](screenshots/ubuntu/01-environment.png) |
| 2 | Каталог `wordpress-shotyk`, `git init`, `.gitignore`, `.dockerignore`, `.env` (паролі приховано) | ![](screenshots/ubuntu/02-project-structure.png) |
| 3 | `openssl genrsa` та `openssl req -x509` створюють самопідписаний сертифікат на 365 днів | ![](screenshots/ubuntu/03-openssl-certificate.png) |
| 4 | `nginx.conf` | ![](screenshots/ubuntu/04a-nginx-conf.png) ![](screenshots/ubuntu/04b-nginx-conf.png) |
| 5 | `docker-compose.yml` | ![](screenshots/ubuntu/05a-docker-compose-yml.png) ![](screenshots/ubuntu/05b-docker-compose-yml.png) |
| 6 | `docker compose down -v` → `up -d` → `ps`, томи й мережа | ![](screenshots/ubuntu/06-docker-compose-up.png) |
| 7 | Перевірки: HTTP 301 → HTTPS, сторінка встановлення, TLS 1.2 з нашим сертифікатом | ![](screenshots/ubuntu/07-curl-checks.png) |
| 8 | Firefox попереджає про самопідписаний сертифікат (`MOZILLA_PKIX_ERROR_SELF_SIGNED_CERT`) | ![](screenshots/ubuntu/08-firefox-self-signed-warning.png) |
| 9 | Вибір мови | ![](screenshots/ubuntu/09-wordpress-language.png) |
| 10 | Назва сайту, користувач `shotyk_admin` (не `admin`), надійний пароль, заборона індексації | ![](screenshots/ubuntu/10-wordpress-setup-form.png) |
| 11 | WordPress встановлено | ![](screenshots/ubuntu/11-wordpress-installed.png) |
| 12 | Вхід | ![](screenshots/ubuntu/12-wordpress-login.png) |
| 13 | Створення власної сторінки в редакторі Gutenberg | ![](screenshots/ubuntu/13-wordpress-page-editor.png) |
| 14 | Сторінку опубліковано | ![](screenshots/ubuntu/14-wordpress-page-published.png) |
| 15–17 | Сторінка як головна сторінка сайту | ![](screenshots/ubuntu/15-wordpress-site-frontpage.png) ![](screenshots/ubuntu/16-wordpress-page-content.png) ![](screenshots/ubuntu/17-wordpress-page-content-2.png) |
| 18 | Мережа з боку Ubuntu: інтерфейси NAT, `office`, host-only | ![](screenshots/ubuntu/18-network-ubuntu.png) |

Зауваження під час роботи. Перша ініціалізація MySQL перервалася через вимкнення VM, і том `dbdata` залишився пошкодженим (`InnoDB: data files are corrupt`). Як і радить методичка, його довелося видалити командою `docker compose down -v` і запустити стек наново.

## 5. Хід роботи на Windows 10 (хост, Docker Desktop)

| # | Крок | Знімок |
|---|---|---|
| 0 | Невдала спроба у VM Win10Client: компоненти WSL/VMP увімкнені, але гіпервізор Hyper-V не стартує | ![](screenshots/windows/00-vm-no-hyperv.png) |
| 1 | Середовище: Windows 10, гіпервізор, WSL 3.0.5, Docker Desktop з бекендом WSL2, host-only адаптер 192.168.56.1 | ![](screenshots/windows/01-host-environment.png) |
| 2 | Запис у `C:\Windows\System32\drivers\etc\hosts` (через UAC) і `ping oleg-shotyk.local` | ![](screenshots/windows/02-hosts-file.png) |
| 3 | `mkdir wordpress-shotyk`, `nginx-conf`, `git init`, `.gitignore`, `.dockerignore` | ![](screenshots/windows/03-project-structure.png) |
| 4 | `.env` (паролі приховано) і `git status --ignored`, який показує, що `.env` ігнорується | ![](screenshots/windows/04-env-file.png) |
| 5 | OpenSSL: ключ і сертифікат | ![](screenshots/windows/05-openssl-certificate.png) |
| 6 | `nginx.conf` | ![](screenshots/windows/06a-nginx-conf.png) ![](screenshots/windows/06b-nginx-conf.png) |
| 7 | `docker-compose.yml` | ![](screenshots/windows/07a-docker-compose-yml.png) ![](screenshots/windows/07b-docker-compose-yml.png) |
| 8 | `docker compose up -d` (з завантаженням образів), `ps`, томи, мережа, журнал MySQL | ![](screenshots/windows/08a-docker-compose-up.png) ![](screenshots/windows/08b-docker-compose-ps.png) |
| 9 | Перевірки `curl.exe`: 301 → HTTPS, сторінка встановлення | ![](screenshots/windows/09-curl-checks.png) |
| 10 | Chrome: `NET::ERR_CERT_AUTHORITY_INVALID` | ![](screenshots/windows/10-chrome-self-signed-warning.png) |
| 12–15 | Форма налаштування, встановлення, вхід, консоль адміністратора | ![](screenshots/windows/12-wordpress-setup-form.png) ![](screenshots/windows/13-wordpress-installed.png) ![](screenshots/windows/14-wordpress-login.png) ![](screenshots/windows/15-wordpress-dashboard.png) |
| 16–17 | Власна сторінка: редактор коду Gutenberg і збережена сторінка | ![](screenshots/windows/16-wordpress-page-code-editor.png) ![](screenshots/windows/17-wordpress-page-saved.png) |
| 18 | «Settings → Reading»: сторінка як головна | ![](screenshots/windows/18-wordpress-reading-settings.png) |
| 19–20 | Сайт | ![](screenshots/windows/19-wordpress-site-frontpage.png) ![](screenshots/windows/20-wordpress-page-content.png) |
| 21 | Мережа: з Windows доступні і ping до Ubuntu, і HTTPS-сайт на Ubuntu (192.168.56.20), і локальний сайт | ![](screenshots/windows/21-network-host-to-ubuntu.png) |

## 6. Поєднання машин у мережу

- **Host-only мережа VirtualBox `192.168.56.0/24`.** Windows-хост має адресу 192.168.56.1, `UbuntuVM` 192.168.56.20 (статична адреса через NetworkManager, з'єднання `hostonly-56`). З Windows до Ubuntu проходять і ping, і HTTPS-запит, який повертає сторінку Ubuntu-WordPress (знімок W21).
- **Внутрішня мережа `office` `192.168.1.0/24`.** До неї підключені `UbuntuVM` (192.168.1.20) і `Win10Client` (192.168.1.2).
- **Зворотний напрямок.** Спершу з Ubuntu до Windows-хоста ping і HTTPS не проходили (знімок U18): брандмауер Windows блокував вхідні з'єднання на host-only адаптері, бо віднів його до «загальнодоступних» мереж. Змінити профіль цієї мережі в «Параметрах» неможливо: host-only адаптер без шлюзу Windows вважає неідентифікованою мережею. Тому адміністратор хоста додав два вхідні правила брандмауера лише для підмережі 192.168.56.0/24: ICMPv4 і TCP 80, 443. Після цього ping з Ubuntu (192.168.56.20) до Windows (192.168.56.1) проходить без втрат.

## 7. Висновок: відмінності між Ubuntu і Windows

1. **Як працює Docker.** В Ubuntu контейнери запускаються прямо на ядрі хоста: Docker Engine — це звичайна служба Linux, а `docker compose` — плагін CLI. У Windows Linux-контейнери потребують Linux-ядра, тому Docker Desktop запускає легку VM WSL2 на гіпервізорі Hyper-V. Через це Windows-варіант залежить від апаратної віртуалізації й гіпервізора Windows.
2. **Віртуалізація всередині VirtualBox.** Через п. 1 Docker Desktop не запрацював у VM `Win10Client`. VirtualBox не забезпечує вкладений Hyper-V для гостьової Windows. Із паравіртуалізацією Hyper-V від VirtualBox гостьова Windows не запускає власний гіпервізор (`HCS_E_HYPERV_NOT_INSTALLED`). Без неї гіпервізор Windows намагається стартувати й ламає завантаження: довелося відновлювати систему через WinRE. Ubuntu у тій самій VirtualBox запускає контейнери без жодних додаткових умов. Windows-частину зрештою виконано на фізичному хості з увімкненим Hyper-V. Побічний ефект: поки Hyper-V активний, VirtualBox працює поверх нього в повільнішому режимі сумісності.
3. **Встановлення інструментів.** В Ubuntu Docker, OpenSSL і git ставляться з репозиторіїв (`apt`), а OpenSSL входить до системи. У Windows Docker Desktop — окремий інсталятор із ліцензійною угодою. OpenSSL не вбудований: його ставлять окремо (slproweb.com, як у методичці, що й зроблено у VM Win10Client) або беруть із Git for Windows (так зроблено на хості).
4. **Права й системні файли.** В Ubuntu досить `sudo` для `/etc/hosts` і `docker`. У Windows зміна `hosts` і встановлення Docker Desktop вимагають підтвердження UAC, а користувача треба додати до групи `docker-users`.
5. **Команди й оболонка.** Синтаксис PowerShell відрізняється від bash: `curl` у PowerShell — це псевдонім `Invoke-WebRequest`, тож потрібен `curl.exe`; замість `grep` використовується `Select-String`; `openssl s_client` чекає на stdin інакше. Шляхи пишуться з `\`, а в профілі користувача можуть бути кириличні символи, проте Docker Desktop монтує такі каталоги (`./nginx-conf`) без проблем.
6. **Мережа.** В Ubuntu опубліковані порти 80/443 одразу доступні ззовні. У Windows їх пропускає Docker Desktop (`com.docker.backend`), але вхідний трафік на нових адаптерах додатково фільтрує брандмауер Windows, як видно з розд. 6.
7. **Однакове.** Файли `docker-compose.yml`, `nginx.conf` і `.env` на обох ОС ідентичні. Образи ті самі, `linux/amd64`. Поведінка стеку (редирект 301, TLS, встановлення WordPress, сторінки) не відрізняється. У цьому головна перевага Docker Compose: опис інфраструктури переноситься між ОС без змін, різниться лише спосіб, яким ОС надає Linux-ядро для контейнерів.
