# Настройка SA Architect Guide на Supabase

Цель: реальный логин для команды (~15 человек + 2 ревьюера) без своего сервера и без деплоя.
Время на настройку: ~20–30 минут, один раз.

Никакого SSO и корпоративной почты — логины синтетические (просто строки-идентификаторы).
Ничего корпоративного в стороннем сервисе не хранится.

---

## Шаг 1. Создать проект Supabase (2 клика)

1. Зайди на supabase.com → Sign up (можно через GitHub).
2. New project → задай имя (напр. `sa-guide`), придумай пароль БД, выбери регион поближе.
3. Подожди ~2 минуты, пока проект поднимется.

## Шаг 2. Создать таблицы и правила доступа (RLS)

Открой в проекте раздел **SQL Editor** → New query → вставь и выполни весь блок ниже.
Это создаёт таблицы и правила «писать можно только в свою запись, ревьюер видит всех».

```sql
-- Таблица прогресса: одна строка на пользователя
create table public.progress (
  user_id     uuid primary key references auth.users on delete cascade,
  name        text,
  checked     jsonb  default '{}'::jsonb,
  evidence    jsonb  default '{}'::jsonb,
  status      jsonb  default '{}'::jsonb,
  review      jsonb  default '{}'::jsonb,
  baseline    jsonb,
  baseline_pct int,
  started     bigint,
  updated     bigint
);

-- Таблица ревьюеров: кто имеет права руководителя
create table public.reviewers (
  user_id uuid primary key references auth.users on delete cascade
);

-- Включаем Row Level Security
alter table public.progress  enable row level security;
alter table public.reviewers enable row level security;

-- Хелпер: текущий пользователь — ревьюер?
create or replace function public.is_reviewer()
returns boolean language sql security definer stable as $$
  select exists (select 1 from public.reviewers r where r.user_id = auth.uid());
$$;

-- progress: читать свою строку или все (если ревьюер)
create policy "progress_select" on public.progress
  for select using ( user_id = auth.uid() or public.is_reviewer() );

-- progress: вставлять только свою строку
create policy "progress_insert" on public.progress
  for insert with check ( user_id = auth.uid() );

-- progress: обновлять свою строку или любую (если ревьюер)
create policy "progress_update" on public.progress
  for update using ( user_id = auth.uid() or public.is_reviewer() );

-- reviewers: любой авторизованный может прочитать (чтобы клиент понял свою роль)
create policy "reviewers_select" on public.reviewers
  for select using ( auth.role() = 'authenticated' );
```

## Шаг 3. Отключить самостоятельную регистрацию

Чтобы аккаунты заводил только ты:
- **Authentication → Providers → Email**: оставь включённым.
- **Authentication → Sign In / Providers** (или Settings): выключи «Allow new users to sign up» (Disable signups).
Так посторонние не зарегистрируются сами.

## Шаг 4. Завести аккаунты команды

**Authentication → Users → Add user** (для каждого человека):
- Email: синтетический логин, напр. `ivan.petrov@sa.team` (реальный ящик не нужен).
- Password: задай начальный пароль, передашь человеку.
- Включи **Auto Confirm User** (чтобы не требовалось подтверждение почты).

Заведи всех ~15 аналитиков и 2 ревьюеров так же.

## Шаг 5. Назначить ревьюеров

После создания пользователей у каждого есть **User UID** (виден в списке Users, можно скопировать).
Для двух ревьюеров вставь их UID в таблицу `reviewers` через SQL Editor:

```sql
insert into public.reviewers (user_id) values
  ('UID-первого-ревьюера'),
  ('UID-второго-ревьюера');
```

Кто в `reviewers` — попадает в режим руководителя. Остальные — в режим аналитика.

## Шаг 6. Вставить ключи в приложение

В файле `sa_architect_guide.html`, в самом верху `<script>`, замени плейсхолдеры:

```javascript
const SUPABASE_URL='https://ВАШ-ПРОЕКТ.supabase.co';
const SUPABASE_ANON_KEY='ВАШ-ANON-КЛЮЧ';
```

Где взять: **Project Settings → API**:
- `Project URL` → в `SUPABASE_URL`
- `anon public` ключ → в `SUPABASE_ANON_KEY`

> anon-ключ безопасно держать в браузере — доступ защищает RLS, а не секретность ключа.
> НИКОГДА не вставляй сюда `service_role` ключ.

## Шаг 7. Захостить файл (без сервера)

Любой из вариантов, все бесплатные:

**Netlify Drop (проще всего):**
1. Переименуй файл в `index.html`.
2. Зайди на app.netlify.com/drop → перетащи файл в окно.
3. Получишь ссылку вида `https://что-то.netlify.app` → раздай команде.

**Cloudflare Pages / GitHub Pages** — тоже подходят, если привычнее.

> Важно: открывать приложение нужно по ссылке хостинга в обычном браузере.
> Внутри Claude оно работать не будет — там нет доступа к сети Supabase.

---

## Как это закрывает твои угрозы

| Угроза | Как закрыто |
|--------|-------------|
| Кто-то заходит под чужим именем | Логин с паролем на сервере Supabase — без пароля под Ирину не зайти |
| Кто-то портит чужую запись | RLS: писать можно только в свою строку (`user_id = auth.uid()`) |
| Кто-то притворяется руководителем | Роль ревьюера — это запись в таблице `reviewers`, которую заводишь только ты |

## Остаточный риск (честно)

Аналитик технически может через прямой API-запрос пометить **свой** этап как принятый (RLS разрешает писать в свою строку целиком). UI этого не даёт, и для доверенной команды это несущественно. Если нужно закрыть и это — выносим решения о приёмке в отдельную таблицу `reviews`, куда пишут только ревьюеры, а статус «принято» вычисляется из неё. Скажи — добавлю SQL и поправлю код.

## Обновление пароля

Если человек забыл пароль — в **Authentication → Users** открой пользователя и задай новый. Сброс по почте не настраиваем (синтетические логины).
