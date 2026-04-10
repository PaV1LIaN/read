Да. Тогда следующий шаг правильный: сначала определить, откуда именно брать статистику, а потом уже подключать её в dashboard.

Сейчас у тебя есть конструктор сайтов, но нет нормального контура аналитики. Значит нужно добавить свой слой сбора статистики.


---

Что именно нужно собирать

Для dashboard тебе нужны такие данные:

По сайту

просмотры сайта

уникальные посетители

количество сессий

средняя длительность сессии

последний визит

активность по дням


По системе

создание сайта

редактирование сайта

публикация страницы

изменение layout

изменение меню

загрузка файлов

входы в админку



---

Как сделать правильно в твоём проекте

Так как проект у тебя лежит в /local, без модулей и без усложнения, я бы делал так:

1. Добавить отдельное хранилище аналитики

Не мешать это с sites.json, pages.json и т.д.

Сделать отдельные файлы в /upload/sitebuilder/:

site_stats.json
site_daily_stats.json
activity_log.json


---

Какие файлы для чего

/upload/sitebuilder/site_stats.json

Агрегированная статистика по сайту:

[
  {
    "siteId": 1,
    "visits": 1200,
    "uniqueVisitors": 430,
    "sessions": 580,
    "avgSessionSeconds": 312,
    "lastVisitAt": "2026-04-10T12:30:00+03:00"
  }
]

/upload/sitebuilder/site_daily_stats.json

Динамика по дням:

[
  {
    "siteId": 1,
    "date": "2026-04-10",
    "visits": 120,
    "uniqueVisitors": 54,
    "sessions": 68,
    "avgSessionSeconds": 298
  }
]

/upload/sitebuilder/activity_log.json

Журнал действий:

[
  {
    "id": 1,
    "type": "page.update",
    "siteId": 1,
    "userId": 1,
    "userLogin": "admin",
    "message": "Обновлена страница Главная",
    "createdAt": "2026-04-10T12:31:00+03:00"
  }
]


---

Откуда брать посещаемость

Есть 2 варианта.

Вариант А — правильный для твоего текущего проекта

Собирать посещаемость через public.php.

То есть каждый раз, когда кто-то открывает:

/local/sitebuilder/public.php?siteId=...

мы:

фиксируем визит

считаем уникальность

считаем сессию

обновляем daily stats


Это самый простой и реалистичный путь.

Вариант Б

Если позже будут красивые URL и реальные сайты вне public.php, тогда статистику надо будет вешать на публичный роутинг/шаблон сайта. Но это следующий этап.

Сейчас бери вариант А.


---

Как считать уникального посетителя

Без излишней сложности:

Минимально рабочий вариант

Использовать ключ:

IP

User-Agent

date

siteId


Например hash:

sha1($siteId . '|' . $ip . '|' . $userAgent . '|' . date('Y-m-d'))

Тогда один пользователь в рамках дня считается уникальным один раз.

Это не идеальная аналитика, но для внутреннего конструктора нормально.


---

Как считать сессию

Тоже без лишней магии:

Упрощённый вариант

Использовать cookie:

SB_SITE_SESSION_<siteId>


Если cookie нет — новая сессия
Если есть и прошло больше 30 минут — новая сессия
Иначе продолжаем текущую

Среднюю длительность можно считать грубо:

сохранять startedAt

сохранять lastSeenAt

потом добавлять разницу


На первом этапе можно даже сделать проще:

считать только visits / unique / sessions

avgSessionSeconds пока брать из накопленного расчёта по cookie-based session tracking



---

Как подключить журнал действий

Очень важно: у тебя уже куча action-ов в API.
Значит нужно просто логировать их централизованно.

Например логировать:

site.create

site.update

page.create

page.updateMeta

page.setStatus

menu.create

menu.update

layout.updateSettings

block.create

block.update

file.upload

file.delete


Это даст блок:

“Последние действия пользователей”

“Сводная активность по системе”



---

Практическая структура в проекте

Я бы добавил так:

/local/sitebuilder/lib/stats.php
/local/sitebuilder/lib/activity.php


---

Что будет в stats.php

Там функции типа:

sb_read_site_stats()
sb_write_site_stats()
sb_read_site_daily_stats()
sb_write_site_daily_stats()

sb_stats_track_visit(int $siteId): void
sb_stats_get_dashboard_summary(): array
sb_stats_get_site_rows(): array
sb_stats_get_popular_sites(): array
sb_stats_get_low_activity_sites(): array
sb_stats_get_daily_chart(int $days = 30): array


---

Что будет в activity.php

sb_read_activity_log()
sb_write_activity_log()
sb_activity_log(string $type, int $siteId, string $message, array $extra = []): void
sb_activity_last(int $limit = 10): array
sb_activity_summary(): array


---

Как подключать это в код

1. В public.php

После определения siteId вызывать:

sb_stats_track_visit($siteId);

2. В API handler-ах

После успешного действия вызывать:

sb_activity_log('page.create', $siteId, 'Создана страница ...');


---

Как отдавать статистику в dashboard

Лучше не тянуть это из site.list, а сделать отдельный action:

dashboard.stats

И отдельный handler:

/local/sitebuilder/api/handlers/dashboard.php

Он будет возвращать всё одним ответом:

{
  "ok": true,
  "summary": {
    "sitesTotal": 4,
    "publishedTotal": 3,
    "draftTotal": 1,
    "visitsTotal": 1540,
    "uniqueVisitorsTotal": 532,
    "avgSessionSeconds": 284
  },
  "chart": [
    {"date":"2026-04-01","visits":40,"uniqueVisitors":20},
    {"date":"2026-04-02","visits":55,"uniqueVisitors":23}
  ],
  "sites": [
    {
      "siteId": 1,
      "name": "Тестовый сайт",
      "slug": "test-site",
      "status": "published",
      "usersCount": 54,
      "visits": 420,
      "uniqueVisitors": 130,
      "avgSessionSeconds": 312,
      "lastActivityAt": "2026-04-10T12:30:00+03:00"
    }
  ],
  "popularSites": [],
  "lowActivitySites": [],
  "lastActions": [],
  "activitySummary": {
    "pageEdits": 12,
    "publishes": 4,
    "uploads": 8,
    "adminLogins": 3
  }
}


---

Что я рекомендую делать по шагам

Шаг 1

Добавить backend сбора статистики:

lib/stats.php

lib/activity.php


Шаг 2

Подключить tracking визитов в public.php

Шаг 3

Подключить логирование действий в API handlers

Шаг 4

Добавить dashboard.stats

Шаг 5

Переделать index.php, чтобы он брал реальные данные из dashboard.stats, а не mock


---

Самый правильный следующий практический шаг

Не распыляться.
Сначала сделать backend статистики и логирования, иначе dashboard останется декоративным.

Значит дальше я должен прислать тебе пакетом:

1. lib/stats.php


2. lib/activity.php


3. что добавить в public.php


4. как расширить api/index.php и сделать dashboard.stats


5. готовый api/handlers/dashboard.php



Напиши: давай backend статистики пакетом