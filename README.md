# ЛогоперРадар — Fleet Tracking System

Веб-приложение для отслеживания местонахождения флота по IMO-кодам с автоматическим определением морского бассейна.

---

## Структура проекта

```
logoper-radar/
├── backend/
│   ├── main.py          # FastAPI приложение (API endpoints)
│   ├── database.py      # SQLite через aiosqlite
│   ├── scraper.py       # Парсинг goradar.ru / myshiptracking.com / vesselfinder.com
│   ├── basin_logic.py   # Логика определения бассейна по портам
│   ├── scheduler.py     # APScheduler — обновление каждые 3 дня
│   ├── auth.py          # JWT-аутентификация
│   └── requirements.txt
├── frontend/
│   ├── index.html       # Публичная страница (без авторизации)
│   └── admin.html       # Панель администратора
├── Dockerfile
├── docker-compose.yml
├── nginx.conf
└── README.md
```

---

## Быстрый старт (Docker)

### 1. Клонировать/скопировать проект на сервер

```bash
scp -r logoper-radar/ user@your-server:/home/user/
```

### 2. Изменить пароль и секрет в docker-compose.yml

```yaml
environment:
  - ADMIN_LOGIN=admin
  - ADMIN_PASSWORD=ВАШ_ПАРОЛЬ      # ← обязательно сменить!
  - SECRET_KEY=длинная-случайная-строка-минимум-32-символа
```

### 3. Запустить

```bash
cd logoper-radar
docker-compose up -d --build
```

Приложение будет доступно на `http://ВАШ_IP:8000`

### 4. Опционально: nginx + SSL

```bash
# Установить nginx
apt install nginx

# Скопировать конфиг
cp nginx.conf /etc/nginx/sites-available/logoper-radar
ln -s /etc/nginx/sites-available/logoper-radar /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx

# SSL через certbot
apt install certbot python3-certbot-nginx
certbot --nginx -d your-domain.com
```

---

## Запуск без Docker (локально / на VPS)

```bash
cd backend

# Создать виртуальное окружение
python3 -m venv venv
source venv/bin/activate

# Установить зависимости
pip install -r requirements.txt

# Задать переменные окружения
export ADMIN_LOGIN=admin
export ADMIN_PASSWORD=radar2026
export SECRET_KEY=my-secret-key-change-me
export DB_PATH=./radar.db

# Запустить
uvicorn main:app --host 0.0.0.0 --port 8000
```

---

## Использование

### Публичная страница
- `http://ВАШ_ДОМЕН/` — просмотр флота, фильтрация, экспорт XLS

### Вход в админ-панель
- Нажать кнопку «Войти» в правом верхнем углу
- Ввести логин/пароль
- После входа — переход на `/admin.html`

### Функции администратора

| Действие | Как |
|---|---|
| Добавить одно судно | Поле «Добавить судно» → ввести IMO → Enter |
| Импорт списка IMO | Скрепка в поисковой строке → TXT файл |
| Удалить судно | Кнопка × в строке таблицы |
| Массовое удаление | Кнопка «Массовое удаление» или скрепка рядом |
| Экспорт XLS | Кнопка «Экспорт XLS» |
| Ручное обновление | Кнопка «Обновить» (запускает парсинг сейчас) |

### Формат TXT файла для импорта/удаления
```
9473626
9512834
1234567
```
Просто список IMO кодов (7 цифр каждый), по одному на строку.

---

## Логика определения бассейна

| Порты захода | Бассейн | Статус |
|---|---|---|
| Владивосток / Находка + иностранные | ДВ | Регулярный |
| Владивосток + Камчатка/Сахалин, только РФ | ДВ каботаж | Каботаж |
| Нет портов РФ | ДВ без РФ | Транзит |
| СПб / Калининград + иностранные | Балтийский | Регулярный |
| Только СПб ↔ Калининград | Балтика каботаж | Каботаж |
| Новороссийск / Туапсе / Тамань | Азово-Черноморский | Регулярный |
| Только порты РФ (разные) | Каботаж | Каботаж |

---

## Расписание обновлений

- Автоматически: каждые **3 дня**
- При запуске приложения: если с последнего обновления прошло > 3 дней — запускается сразу
- Вручную: кнопка «Обновить» в админ-панели

---

## API Endpoints

### Публичные
- `GET /api/vessels` — список всех судов
- `GET /api/stats` — статистика по бассейнам
- `GET /api/vessels/export` — скачать XLSX

### Административные (требуют JWT)
- `POST /api/login` — авторизация `{login, password}`
- `POST /api/admin/vessels` — добавить судно `{imo}`
- `POST /api/admin/vessels/import` — импорт TXT файла
- `DELETE /api/admin/vessels/{imo}` — удалить судно
- `POST /api/admin/vessels/delete-bulk` — массовое удаление
- `POST /api/admin/refresh` — запустить обновление сейчас
- `GET /api/admin/vessels/export` — экспорт XLSX (auth)

---

## Источники данных (бесплатные, без API-ключей)

1. **goradar.ru** — расписания контейнерных линий, порты захода (приоритет)
2. **myshiptracking.com** — позиция, последний порт, назначение (резерв)
3. **vesselfinder.com** — название, флаг, тип судна (резерв)

---

## Переменные окружения

| Переменная | По умолчанию | Описание |
|---|---|---|
| `ADMIN_LOGIN` | `admin` | Логин администратора |
| `ADMIN_PASSWORD` | `radar2026` | Пароль (обязательно сменить!) |
| `SECRET_KEY` | *(строка)* | Ключ подписи JWT (сменить!) |
| `TOKEN_EXPIRE_HOURS` | `72` | Срок действия токена (часы) |
| `DB_PATH` | `radar.db` | Путь к SQLite базе данных |
