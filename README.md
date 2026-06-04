В итоге делаем так:

Главная идея

Шаблон сайта — это сохранённая копия структуры сайта:

сайт
страницы
вложенность страниц
блоки
layout-зоны
меню
настройки оформления

Но файлы диска не копируем.
Если в шаблоне есть блок Диск, то в новом сайте этот блок будет, но папка диска будет новая/пустая.


---

Главное правило по правам

Создавать / редактировать / удалять шаблоны может только администратор Битрикса:

global $USER;
$USER->IsAdmin()

Не OWNER, не ADMIN сайта, не EDITOR.


---

Что нужно сделать по шагам

Шаг 1. Добавить проверку администратора Битрикса

В файл:

/local/sitebuilder/lib/helpers.php

добавить функции:

sb_is_bitrix_admin()
sb_require_bitrix_admin()

Они будут защищать API шаблонов.


---

Шаг 2. Создать хранилище шаблонов

Нужен новый файл:

/local/sitebuilder/lib/TemplateRepository.php

Он будет работать с шаблонами:

получить список шаблонов
получить шаблон по ID
создать шаблон
обновить шаблон
удалить шаблон

Если у тебя сейчас часть проекта уже в PostgreSQL, лучше хранить шаблоны в таблице.
Если пока часть ещё в JSON, можно временно сделать:

/upload/sitebuilder/templates.json


---

Шаг 3. Создать сервис шаблонов

Нужен файл:

/local/sitebuilder/lib/SiteTemplateService.php

В нём будет логика:

создать шаблон из текущего сайта
создать новый сайт из шаблона
очистить disk-настройки при копировании
заменить старые pageId на новые pageId
заменить старые blockId на новые blockId


---

Шаг 4. Создать API-обработчик шаблонов

Нужен файл:

/local/sitebuilder/api/handlers/template.php

Действия:

template.list
template.get
template.create
template.delete
template.createSite

Где:

template.create — только Bitrix admin
template.delete — только Bitrix admin
template.createSite — можно разрешить пользователям, которые имеют право создавать сайты


---

Шаг 5. Подключить обработчик в API

В файле:

/local/sitebuilder/api/index.php

надо подключить:

require __DIR__ . '/handlers/template.php';


---

Шаг 6. Добавить кнопку в редактор сайта

В:

/local/sitebuilder/editor.php

добавить кнопку:

Сохранить как шаблон

Но показывать её только если:

$USER->IsAdmin()


---

Шаг 7. Добавить интерфейс на главной странице

В:

/local/sitebuilder/index.php

добавить блок:

Шаблоны сайтов

Там будут карточки шаблонов:

Название шаблона
Описание
Дата создания
[Создать сайт]


---

Что лучше сделать первым

Начинаем с backend, чтобы не ломать интерфейс.

Первым делом нужно сделать:

1. helpers.php — проверка Bitrix admin
2. TemplateRepository.php
3. SiteTemplateService.php
4. api/handlers/template.php
5. подключение template.php в api/index.php

После этого уже добавим кнопки в интерфейс.


---

Что мне нужно от тебя сейчас

Пришли текущие файлы:

/local/sitebuilder/api/index.php
/local/sitebuilder/api/handlers/site.php
/local/sitebuilder/lib/helpers.php
/local/sitebuilder/lib/SiteRepository.php

И напиши, есть ли у тебя такие файлы:

/local/sitebuilder/lib/PageRepository.php
/local/sitebuilder/lib/BlockRepository.php
/local/sitebuilder/api/handlers/template.php

Если их нет — так и напиши: нет. Тогда я сделаю вариант под твою текущую структуру.