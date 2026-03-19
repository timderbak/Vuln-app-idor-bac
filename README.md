# 🏥 MedClinic API — Vulnerable by Design

REST API медицинской клиники с **23 намеренно заложенными уязвимостями** — 11 IDOR и 12 BAC.

> ⚠️ **Это учебное приложение.** Все уязвимости намеренные. Не деплоить в продакшен.

---

## 🚀 Быстрый старт

```bash
git clone https://github.com/timderbak/vuln-app-idor-bac.git
cd vuln-app-idor-bac

python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

uvicorn app.main:app --host 0.0.0.0 --port 8000
```

База данных заполняется тестовыми данными при первом запуске.

## 📚 Документация

- **Swagger UI**: http://localhost:8000/docs
- **ReDoc**: http://localhost:8000/redoc

## 🏗️ Стек

- FastAPI 0.104
- SQLite + SQLAlchemy 2.0
- JWT + API Key аутентификация
- Python 3.9+

## 👥 Тестовые аккаунты

| Email | Пароль | Роль |
|-------|--------|------|
| `john@patient.com` | `patient123` | patient |
| `jane@patient.com` | `patient123` | patient |
| `mike@patient.com` | `patient123` | patient |
| `sarah@doctor.com` | `doctor123` | doctor |
| `james@doctor.com` | `doctor123` | doctor |
| `anna@nurse.com` | `nurse123` | nurse |
| `admin@clinic.com` | `admin123` | admin |

## 🔑 Аутентификация

```bash
# Получить токен
curl -X POST http://localhost:8000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"john@patient.com","password":"patient123"}'

# Использовать токен
curl http://localhost:8000/api/patients/ \
  -H "Authorization: Bearer <токен>"
```

---

## 🎯 Карта уязвимостей

### IDOR-уязвимости (I1–I11)

| Код | Сложность | Эндпоинт | Описание |
|-----|-----------|----------|----------|
| **I1** | 🟢 Easy | `GET /api/patients/{id}` | **Горизонтальный IDOR** — просмотр чужого профиля пациента |
| **I2** | 🟢 Easy | `PUT /api/patients/{id}` | **Горизонтальный IDOR** — изменение чужого профиля пациента |
| **I3** | 🟢 Easy | `GET /api/appointments/{id}` | **Горизонтальный IDOR** — просмотр чужих записей к врачу |
| **I4** | 🟢 Easy | `DELETE /api/appointments/{id}` | **Горизонтальный IDOR** — отмена чужих записей к врачу |
| **I5** | 🟢 Easy | `GET /api/prescriptions/{id}` | **Горизонтальный IDOR** — просмотр чужих рецептов |
| **I6** | 🟢 Easy | `GET /api/records/{id}` | **Горизонтальный IDOR** — просмотр чужих медицинских записей |
| **I7** | 🟡 Medium | `GET /api/files/{id}/download` | **IDOR файлов** — скачивание чужих медицинских документов |
| **I8** | 🟡 Medium | `GET /api/prescriptions/?patient_id=X` | **IDOR через фильтр** — получение рецептов любого пациента через query-параметр |
| **I9** | 🟡 Medium | `GET /api/records/?patient_id=X` | **IDOR через фильтр** — получение медкарт любого пациента через query-параметр |
| **I10** | 🟢 Easy | `GET /api/admin/users/{id}` | **Вертикальный IDOR** — пациент получает доступ к админ-данным пользователей (включая API-ключи) |
| **I11** | 🟢 Easy | `GET /api/patients/` | **Перечисление ID** — ответ содержит последовательные ID, позволяя перебирать объекты |

### BAC-уязвимости (B1–B12)

| Код | Сложность | Эндпоинт | Описание |
|-----|-----------|----------|----------|
| **B1** | 🟢 Easy | `GET /api/patients/` | **Отсутствие проверки роли** — любой аутентифицированный пользователь видит все профили пациентов |
| **B2** | 🟢 Easy | `POST /api/prescriptions/` | **Отсутствие проверки роли** — пациент может создавать рецепты (должен только врач) |
| **B3** | 🟢 Easy | `GET /api/admin/users` | **Forced browsing** — админ-эндпоинт вообще без аутентификации; утечка данных и API-ключей |
| **B4** | 🟡 Medium | `GET /api/admin/stats` | **Weak API Key check** — принимает ЛЮБОЙ валидный API-ключ, а не только админский |
| **B5** | 🟡 Medium | `POST /api/appointments/` | **Подмена параметров** — можно создать запись от имени другого пациента через `patient_id` |
| **B6** | 🟡 Medium | `POST /api/records/` | **Подмена параметров** — создание медзаписи от имени другого врача |
| **B7** | 🟢 Easy | `DELETE /api/files/{id}` | **Отсутствие аутентификации** — удаление файлов без авторизации (GET при этом требует токен) |
| **B8** | 🟡 Medium | `PUT /api/prescriptions/{id}` | **Отсутствие проверки роли** — пациент может менять статус своего рецепта |
| **B9** | 🟢 Easy | `POST /api/auth/register` | **Mass Assignment** — при регистрации можно передать `role: "admin"` и получить привилегии |
| **B10** | 🟡 Medium | `PUT /api/patients/{id}` | **Mass Assignment** — через обновление профиля можно поменять `role` и `user_id` |
| **B11** | 🟡 Medium | `GET /api/records/` | **Отсутствие проверки роли** — пациент видит ВСЕ медицинские записи системы |
| **B12** | 🟡 Medium | `PUT /api/admin/users/{id}` | **Обход защиты** — `require_role_weak` содержит ошибку в сравнении регистра ролей |

---

## 📁 Структура проекта

```
vuln-app-idor-bac/
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI-приложение, lifespan, CORS, лендинг
│   ├── auth.py               # JWT + API Key аутентификация  
│   ├── database.py           # SQLAlchemy engine & session
│   ├── models.py             # ORM-модели (User, Patient, Appointment и т.д.)
│   ├── seed.py               # Заполнение БД тестовыми данными
│   └── routers/
│       ├── __init__.py
│       ├── auth_router.py    # Логин, регистрация, обновление токена
│       ├── patients.py       # CRUD профилей пациентов
│       ├── appointments.py   # Управление записями к врачу
│       ├── prescriptions.py  # Управление рецептами
│       ├── medical_records.py # Управление медицинскими записями
│       ├── files.py          # Загрузка/скачивание файлов
│       └── admin.py          # Админ-панель
├── uploads/                  # Загруженные медицинские документы
├── requirements.txt
├── .gitignore
└── README.md
```

---

## 🗃️ Модели данных

| Модель | Описание | Ключевые поля |
|--------|----------|---------------|
| **User** | Пользователь системы | `id`, `email`, `name`, `role`, `api_key` |
| **PatientProfile** | Профиль пациента | `user_id`, `blood_type`, `allergies`, `insurance_number` |
| **Appointment** | Запись к врачу | `patient_id`, `doctor_id`, `date`, `status`, `diagnosis` |
| **Prescription** | Рецепт (UUID ID) | `patient_id`, `doctor_id`, `medication`, `dosage`, `status` |
| **MedicalRecord** | Медицинская запись | `patient_id`, `doctor_id`, `record_type`, `result` |
| **File** | Загруженный файл | `owner_id`, `filename`, `file_path`, `file_type` |

### Роли и права

| Роль | Описание | 
|------|----------|
| `patient` | Просмотр своих данных, запись к врачу |
| `doctor` | Пациенты, рецепты, медзаписи |
| `nurse` | Просмотр данных пациентов, помощь с записями |
| `receptionist` | Управление записями к врачу |
| `admin` | Полный доступ к системе |

---

## 🧪 Примеры эксплуатации

### 1. Горизонтальный IDOR (I1)

```bash
# Логин как пациент John (user_id=1)
TOKEN=$(curl -s -X POST http://localhost:8000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"john@patient.com","password":"patient123"}' | jq -r .access_token)

# Просмотр ЧУЖОГО профиля (Jane, patient_id=2)
curl http://localhost:8000/api/patients/2 \
  -H "Authorization: Bearer $TOKEN"
```

### 2. Mass Assignment — регистрация как admin (B9)

```bash
curl -X POST http://localhost:8000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"hacker@evil.com","name":"Hacker","password":"hack123","role":"admin"}'
```

### 3. Forced Browsing — админ без авторизации (B3)

```bash
# Доступ к списку ВСЕХ пользователей без логина
curl http://localhost:8000/api/admin/users
```

### 4. IDOR файлов (I7)

```bash
# Скачивание чужого медицинского документа
curl http://localhost:8000/api/files/1/download \
  -H "Authorization: Bearer $TOKEN"
```

---

## 📝 Лицензия

Проект предназначен **исключительно для образовательных целей**. Используйте ответственно.
