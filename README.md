# GS Transfer — GitHub Pages + Supabase

GS Transfer теперь полностью работает без отдельного Node.js/VPS backend:

- `site/` — статический сайт, публикуется через GitHub Pages;
- Supabase Postgres — метаданные файлов;
- Supabase Storage — сами файлы;
- Supabase Edge Function — безопасная серверная часть: создание загрузки, завершение загрузки, проверка срока жизни и выдача временных ссылок;
- большие файлы загружаются через TUS resumable upload с прогрессом.

Дизайн сайта не менялся.

## 1. Создать проект Supabase

Создайте проект в Supabase.

Важно: бесплатный план Supabase ограничивает максимальный размер одного файла 50 MB. Для файлов до 20 GB нужен тариф/настройки Storage, где такой размер разрешён. Supabase рекомендует resumable/TUS для больших файлов.

## 2. Выполнить SQL

Откройте **SQL Editor** в Supabase и выполните:

```text
supabase/schema.sql
```

Скрипт создаёт:

- таблицу `public.files`;
- RLS-политику для чтения только активных файлов;
- приватный Storage bucket `files`;
- разрешение на загрузку объектов для анонимного клиента.

## 3. Установить Supabase CLI

После установки CLI выполните из корня проекта:

```bash
supabase login
supabase link --project-ref ВАШ_PROJECT_REF
supabase functions deploy api --no-verify-jwt
```

Edge Function находится здесь:

```text
supabase/functions/api/index.ts
```

`--no-verify-jwt` нужен, потому что публичный сайт должен уметь вызывать функцию без регистрации пользователей. Функция сама не выдаёт service-role ключ клиенту.

## 4. Заполнить config.js

Откройте:

```text
site/config.js
```

И укажите:

```js
window.GS_CONFIG = {
  SUPABASE_URL: 'https://ВАШ_PROJECT_REF.supabase.co',
  SUPABASE_ANON_KEY: 'ВАШ_PUBLISHABLE_ИЛИ_ANON_KEY'
};
```

Используйте только публичный ключ Supabase (publishable/anon). **Service role key нельзя помещать в `site/` или GitHub.**

## 5. Загрузить на GitHub

Структура проекта:

```text
.
├── .github/workflows/pages.yml
├── site/
│   ├── index.html
│   ├── config.js
│   └── .nojekyll
└── supabase/
    ├── config.toml
    ├── schema.sql
    └── functions/api/index.ts
```

Создайте репозиторий и отправьте проект в `main`:

```bash
git add .
git commit -m "Prepare GS Transfer for GitHub Pages and Supabase"
git push origin main
```

Затем в GitHub:

**Settings → Pages → Build and deployment → Source → GitHub Actions**.

Workflow уже находится в:

```text
.github/workflows/pages.yml
```

После успешного Actions сайт будет доступен по адресу GitHub Pages.

## Как теперь работает загрузка

1. GitHub Pages отдаёт HTML/JS.
2. Сайт вызывает Supabase Edge Function.
3. Edge Function создаёт запись о файле и временный signed upload token.
4. Браузер отправляет файл напрямую в Supabase Storage через TUS.
5. После окончания загрузки Edge Function отмечает файл как `done`.
6. При открытии ссылки Edge Function проверяет срок хранения и выдаёт временный signed URL.
7. Сам файл не проходит через GitHub Pages и не хранится в репозитории.

Это позволяет оставить GitHub Pages полностью статическим.

## Локальная проверка сайта

Можно открыть `site/index.html` через любой локальный static server. Например:

```bash
python -m http.server 8080 --directory site
```

Открыть:

```text
http://localhost:8080
```

## Важные настройки

- Срок жизни ссылки: 7 дней — задаётся в Edge Function `TTL_DAYS`.
- Максимальный размер: 20 GB — `MAX_FILE`.
- Для больших файлов используется TUS с чанками 6 MB.
- Storage bucket приватный.
- Service role key находится только на стороне Supabase Edge Function.
- GitHub Pages не требует Node.js backend.
