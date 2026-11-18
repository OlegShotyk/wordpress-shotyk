# wordpress-shotyk

Розрахунково-графічна робота з дисципліни «Операційні системи» (НУ «Львівська політехніка», кафедра САП, 2026).

**Тема:** розгортання WordPress-аплікації за допомогою Docker Compose.
**Виконав:** Олег Шотик.

Стек: MySQL 8.0, WordPress 5.1.1 (PHP-FPM, Alpine) і Nginx 1.15.12 з самопідписаним SSL-сертифікатом. Однакову конфігурацію розгорнуто двічі:
- на віртуальній машині VirtualBox з **Ubuntu 26.04** (Docker Engine + Compose plugin);
- на **Windows 10** (Docker Desktop, бекенд WSL2). Спершу спробував у VM Win10Client, але VirtualBox не підтримує вкладений Hyper-V, тому Windows-частину виконано на фізичному хості Windows 10 (подробиці у звіті).

Машини з'єднані host-only мережею VirtualBox `192.168.56.0/24`: хост має адресу 192.168.56.1, Ubuntu 192.168.56.20. Ubuntu також входить до внутрішньої мережі `office` (192.168.1.0/24) разом з VM Win10Client.

Повний звіт з усіма знімками екрана: [REPORT.md](REPORT.md). Версія у Word із титульною сторінкою: [РГР_ОС_Шотик.docx](РГР_ОС_Шотик.docx).

## Структура

```
wordpress-shotyk/
├── docker-compose.yml       # три сервіси: db-shotyk, wordpress-shotyk, webserver-shotyk
├── nginx-conf/
│   └── nginx.conf           # редирект 80→443, SSL, FastCGI до php-fpm
├── .env.example             # шаблон змінних середовища (справжній .env у Git не потрапляє)
├── .gitignore               # .env, приватний ключ, локальні облікові дані
├── .dockerignore
├── page/rgr-page.html       # вміст створеної сторінки WordPress (блоки Gutenberg)
├── screenshots/
│   ├── ubuntu/              # знімки виконання на Ubuntu
│   └── windows/             # знімки виконання на Windows 10
└── REPORT.md                # звіт
```

## Швидкий запуск

```bash
cp .env.example .env            # задати власні паролі
cd nginx-conf
openssl genrsa -out oleg-shotyk.key 2048
openssl req -new -x509 -key oleg-shotyk.key -out oleg-shotyk.crt -days 365
cd ..
docker compose up -d
```

Також треба додати в `hosts` рядок `127.0.0.1 oleg-shotyk.local` і відкрити https://oleg-shotyk.local.

Сертифікат `.crt` лежить у репозиторії для наочності. Приватний ключ `.key` і `.env` навмисно не комітяться.
