Ниже — прикладная схема именно под твой архив, а не абстрактная архитектура.

По факту сейчас у тебя проект плоский: почти всё лежит в корне /local/sitebuilder, а ключевая проблема — слишком толстые файлы:

api.php — 3678 строк

editor.php — 2646 строк

public.php — 1531 строк

layout.php — 1367 строк

index.php — 1143 строки

view.php — 886 строк


Это уже не вопрос “красоты”, а вопрос поддержки: в одном месте смешаны роутинг, права, Disk, JSON-хранилище, бизнес-логика, рендер, обработка POST, JS-диалоги и UI.


---

1. Что сейчас не так

1. api.php перегружен несколькими ролями сразу

Сейчас в нём одновременно:

чтение/запись JSON

работа с site/page/block/menu/template/layout

ACL

Disk

генерация slug

action routing

валидация входных данных

формирование JSON-ответа


То есть это не “API-файл”, а весь backend проекта в одном файле.

2. Повторяются одинаковые низкоуровневые функции

Например:

sb_data_path

sb_read_json_file / sb_read_json

sb_user_access_code

sb_get_role

sb_role_rank


Они есть не только в api.php, но и дублируются в download.php, public.php, view.php.

Это как раз тот код, который реально общий и должен жить в одном месте.

3. Плоская структура в корне

Сейчас файл по имени должен объяснять всё сам:

menu.php

files.php

layout.php

settings.php

public.php

view.php


Для живого проекта это плохо: по дереву нельзя быстро понять, где:

entry point

backend

shared

editor

public rendering

disk

storage


4. В editor.php и layout.php смешан PHP + HTML + огромный inline JS

Особенно это видно по:

openTextDialog

openImageDialog

openButtonDialog

openHeadingDialog

openGalleryDialog

openCardDialog

openCardsDialog


То есть “страница” и “клиентская логика редактора” слеплены в один файл.

5. Нет явной границы между shared-кодом и локальным кодом сущности

Например:

ACL нужен в нескольких местах → это shared

JSON storage нужен в нескольких местах → shared

Disk-обвязка нужна в нескольких местах → shared

логика карточек для layout/editor → не shared для всего проекта, а локально рядом с editor/layout

рендер публичных блоков → локально рядом с public/view



---

2. Нормальная структура для /local/sitebuilder

Я бы делал так:

/local/sitebuilder/
├── index.php
├── editor.php
├── menu.php
├── files.php
├── settings.php
├── layout.php
├── public.php
├── view.php
├── download.php
│
├── api/
│   ├── index.php
│   ├── bootstrap.php
│   ├── helpers.php
│   ├── site.php
│   ├── page.php
│   ├── block.php
│   ├── menu.php
│   ├── access.php
│   ├── file.php
│   ├── template.php
│   └── layout.php
│
├── src/
│   ├── Shared/
│   │   ├── JsonStorage.php
│   │   ├── Response.php
│   │   ├── Request.php
│   │   ├── Slug.php
│   │   └── Utils.php
│   │
│   ├── Auth/
│   │   └── Access.php
│   │
│   ├── Disk/
│   │   ├── DiskStorage.php
│   │   └── DiskRights.php
│   │
│   ├── Repositories/
│   │   ├── SitesRepository.php
│   │   ├── PagesRepository.php
│   │   ├── BlocksRepository.php
│   │   ├── MenusRepository.php
│   │   ├── AccessRepository.php
│   │   ├── TemplatesRepository.php
│   │   └── LayoutsRepository.php
│   │
│   ├── Services/
│   │   ├── SiteService.php
│   │   ├── PageService.php
│   │   ├── BlockService.php
│   │   ├── MenuService.php
│   │   ├── TemplateService.php
│   │   ├── FileService.php
│   │   └── LayoutService.php
│   │
│   ├── PublicPage/
│   │   ├── PublicRenderer.php
│   │   ├── BlockRenderer.php
│   │   ├── Breadcrumbs.php
│   │   └── PageTree.php
│   │
│   └── Editor/
│       ├── BlockConfig.php
│       └── EditorViewData.php
│
├── views/
│   ├── partials/
│   │   ├── header.php
│   │   ├── sidebar.php
│   │   └── alerts.php
│   │
│   ├── editor/
│   │   ├── toolbar.php
│   │   ├── blocks-panel.php
│   │   └── dialogs-container.php
│   │
│   ├── layout/
│   │   ├── toolbar.php
│   │   └── zones.php
│   │
│   └── public/
│       ├── page.php
│       ├── breadcrumbs.php
│       └── menu.php
│
├── assets/
│   ├── css/
│   │   ├── common.css
│   │   ├── index.css
│   │   ├── editor.css
│   │   ├── layout.css
│   │   ├── menu.css
│   │   ├── settings.css
│   │   └── public.css
│   │
│   └── js/
│       ├── shared/
│       │   ├── api.js
│       │   ├── notify.js
│       │   ├── dom.js
│       │   └── modal.js
│       │
│       ├── editor/
│       │   ├── state.js
│       │   ├── blocks-render.js
│       │   ├── block-actions.js
│       │   ├── template-actions.js
│       │   ├── dialogs/
│       │   │   ├── text.js
│       │   │   ├── image.js
│       │   │   ├── button.js
│       │   │   ├── heading.js
│       │   │   ├── columns2.js
│       │   │   ├── gallery.js
│       │   │   ├── spacer.js
│       │   │   ├── card.js
│       │   │   └── cards.js
│       │   └── index.js
│       │
│       ├── layout/
│       │   ├── state.js
│       │   ├── zones-render.js
│       │   ├── block-actions.js
│       │   ├── settings.js
│       │   ├── dialogs/
│       │   │   ├── text.js
│       │   │   ├── image.js
│       │   │   ├── button.js
│       │   │   ├── heading.js
│       │   │   ├── columns2.js
│       │   │   ├── gallery.js
│       │   │   ├── spacer.js
│       │   │   ├── card.js
│       │   │   └── cards.js
│       │   └── index.js
│       │
│       ├── menu/
│       │   └── index.js
│       │
│       ├── files/
│       │   └── index.js
│       │
│       ├── settings/
│       │   └── index.js
│       │
│       └── index/
│           └── index.js
│
└── data/
    ├── sites.json
    ├── pages.json
    ├── blocks.json
    ├── menus.json
    ├── access.json
    ├── templates.json
    └── layouts.json


---

3. Что где должно лежать

Корень /local/sitebuilder

Здесь должны остаться только точки входа:

index.php

editor.php

layout.php

menu.php

files.php

settings.php

public.php

view.php

download.php


Их задача:

подключить bootstrap

собрать данные

отрисовать нужный view

подключить CSS/JS


Они не должны содержать 1000+ строк бизнес-логики.


---

/api

Здесь лежат backend entry point и action handlers.

Должно быть:

разбор action

подключение общего bootstrap

вызов нужного обработчика

возврат JSON


Не должно быть:

дублирования JSON storage

дублирования ACL

дублирования Disk helper

огромной логики всех сущностей в одном файле



---

/src/Shared

Только реально общее:

чтение request

JSON response

slug

общие утилиты

базовая работа с json storage


Сюда не надо складывать всё подряд.


---

/src/Auth

Только права:

sb_user_access_code

sb_get_role

sb_role_rank

sb_require_owner/admin/editor/viewer


Это точно shared, потому что используется в нескольких входных точках.


---

/src/Disk

Вся совместимость с Bitrix Disk:

create folder

upload file

list children

sync rights

ensure site folder

belongs-to-site check


Disk — отдельная зона ответственности, её надо изолировать.


---

/src/Repositories

Только доступ к данным:

sites.json

pages.json

blocks.json

menus.json

access.json

templates.json

layouts.json


Репозиторий должен:

читать

писать

находить

обновлять записи


Он не должен:

делать права

знать про HTTP

рендерить HTML

показывать ошибки пользователю



---

/src/Services

Здесь уже прикладная логика:

создать сайт

удалить страницу

продублировать блок

применить шаблон

обновить layout

загрузить файл


Если логика состоит из 1–2 строк, отдельный service можно не делать. Но там, где есть несколько шагов, проверки, каскадные изменения — service нужен.


---

/src/PublicPage

Только публичный рендер:

рендер блоков

хлебные крошки

дерево страниц

public url

left menu


Этот код не должен лежать в shared, потому что он нужен не всему проекту, а именно публичной части.


---

/assets/js

Только JS. Не надо хранить большие inline-скрипты в editor.php и layout.php.


---

/views

Только PHP-шаблоны и partials. Не нужно делать “MVC ради MVC”, но повторяющиеся куски интерфейса лучше вынести.


---

4. Граница между shared и локальным кодом

Вот самое важное правило для этого проекта.

Выносить в shared / src, если код:

используется минимум в 2–3 местах

не зависит от конкретного экрана

не зависит от конкретной сущности UI

относится к инфраструктуре проекта


Примеры:

sb_data_path

sb_read_json_file

sb_write_json_file

sb_user_access_code

sb_get_role

sb_role_rank

sb_slugify

JSON response helper

Bitrix Disk wrappers



---

Оставлять локально рядом с сущностью, если код:

нужен только editor

нужен только layout

нужен только public render

нужен только menu page

зависит от конкретного DOM / конкретного UI / конкретной структуры блока


Примеры:

cardsNormalizeItem() для JS-конструктора карточек

cardsRenderBuilderItems() для диалога cards

openGalleryDialog() как UI-диалог

рендер конкретного блока в preview editor/layout

логика кнопок “переместить вверх/вниз” в редакторе


Это не shared, даже если хочется “красиво вынести”.


---

5. Как правильно разрезать api.php

Сейчас его надо резать не по строкам, а по ролям.

Что оставить в api/index.php

Только:

подключение bootstrap.php

получение $action

карта action → handler

вызов нужного файла/обработчика

единый try/catch


Пример:

<?php
require __DIR__ . '/bootstrap.php';

$action = (string)($_POST['action'] ?? '');

$map = [
    'site.list' => __DIR__ . '/site.php',
    'site.get' => __DIR__ . '/site.php',
    'site.create' => __DIR__ . '/site.php',
    'site.update' => __DIR__ . '/site.php',
    'site.delete' => __DIR__ . '/site.php',
    'site.setHome' => __DIR__ . '/site.php',

    'page.list' => __DIR__ . '/page.php',
    'page.create' => __DIR__ . '/page.php',
    'page.delete' => __DIR__ . '/page.php',
    'page.duplicate' => __DIR__ . '/page.php',
    'page.updateMeta' => __DIR__ . '/page.php',
    'page.setStatus' => __DIR__ . '/page.php',
    'page.setParent' => __DIR__ . '/page.php',
    'page.move' => __DIR__ . '/page.php',

    'block.list' => __DIR__ . '/block.php',
    'block.create' => __DIR__ . '/block.php',
    'block.update' => __DIR__ . '/block.php',
    'block.delete' => __DIR__ . '/block.php',
    'block.duplicate' => __DIR__ . '/block.php',
    'block.move' => __DIR__ . '/block.php',
    'block.reorder' => __DIR__ . '/block.php',

    'layout.get' => __DIR__ . '/layout.php',
    'layout.updateSettings' => __DIR__ . '/layout.php',
    'layout.block.list' => __DIR__ . '/layout.php',
    'layout.block.create' => __DIR__ . '/layout.php',
    'layout.block.update' => __DIR__ . '/layout.php',
    'layout.block.delete' => __DIR__ . '/layout.php',
    'layout.block.move' => __DIR__ . '/layout.php',
];

if (!isset($map[$action])) {
    json_error('UNKNOWN_ACTION', 400);
}

require $map[$action];


---

Как разложить обработчики

api/site.php

site.list

site.get

site.create

site.update

site.delete

site.setHome


api/page.php

page.list

page.create

page.delete

page.duplicate

page.updateMeta

page.setStatus

page.setParent

page.move


api/block.php

block.list

block.create

block.update

block.delete

block.duplicate

block.move

block.reorder


api/menu.php

menu.list

menu.create

menu.update

menu.delete

menu.setTop

если есть item-операции — тоже сюда


api/template.php

template.list

template.createFromPage

template.applyToPage

template.rename

template.delete


api/access.php

access.list

access.set

access.delete


api/file.php

file.list

file.upload

file.delete


api/layout.php

layout.get

layout.updateSettings

layout.block.*



---

Что вынести из api.php в src сразу

Вот это выносится первым:

JSON storage helpers

ACL

slug

Disk wrappers

layout low-level helpers

menu low-level helpers

site/page/block find helpers



---

6. Как резать большие JS-файлы типа editor.php / editor.dialogs.js

У тебя в editor.php уже видно отдельные естественные куски.

Не надо резать “по 100 строк”

Надо резать по ответственности.


---

Как разложить JS редактора

assets/js/shared/api.js

Общий AJAX-вызов:

export function api(action, data = {}) {
  return new Promise((resolve, reject) => {
    BX.ajax({
      url: '/local/sitebuilder/api/index.php',
      method: 'POST',
      data: { action, ...data },
      dataType: 'json',
      onsuccess: resolve,
      onfailure: reject
    });
  });
}


---

assets/js/shared/notify.js

Общее уведомление.


---

assets/js/editor/state.js

Только состояние редактора:

siteId

pageId

collapsedBlocks

фильтры

текущий список блоков



---

assets/js/editor/blocks-render.js

Только рендер превью блоков:

renderBlocks

buildBlockShell

headingTag

headingAlign

colsGridTemplate

galleryTemplate



---

assets/js/editor/block-actions.js

Операции:

load blocks

delete block

duplicate

move

quick add



---

assets/js/editor/template-actions.js

saveTemplateFromPage

applyTemplateToPage



---

assets/js/editor/dialogs/*.js

Каждый диалог отдельно:

text.js

image.js

button.js

heading.js

gallery.js

cards.js


Это как раз идеальный случай локального кода по назначению.


---

Что точно не надо выносить в shared JS

Например:

openCardsBuilderDialog

cardsNormalizeItem

cardsRenderBuilderItems


Если они используются только в editor/layout-карточках, держи их рядом с карточками.


---

Для layout.php аналогично

Выделить:

assets/js/layout/state.js

assets/js/layout/zones-render.js

assets/js/layout/settings.js

assets/js/layout/block-actions.js

assets/js/layout/dialogs/*.js



---

7. Что вынести в общее, а что оставить локально

Вынести в общее

PHP shared

sb_data_path

sb_read_json_file

sb_write_json_file

sb_slugify

sb_user_access_code

sb_get_role

sb_role_rank

sb_require_*

общие json response helpers

общие request helpers


PHP disk

sb_disk_add_subfolder

sb_disk_upload_file

sb_disk_get_children

sb_disk_common_storage

sb_disk_get_or_create_root

sb_disk_ensure_site_folder

sb_disk_sync_folder_rights

sb_disk_file_belongs_to_site


PHP repository-level

sb_read_sites / write_sites

sb_read_pages / write_pages

sb_read_blocks / write_blocks

sb_read_access / write_access

sb_read_menus / write_menus

sb_read_templates / write_templates

sb_read_layouts / write_layouts



---

Оставить локально

В src/PublicPage

sb_render_block

sb_render_blocks

sb_render_breadcrumbs

sb_render_left_menu_tree

public_page_url

file_url


В layout logic

sb_layout_valid_zone

sb_layout_zone_blocks

sb_layout_zone_set

sb_layout_next_block_id

sb_layout_find_block


Это не shared на весь проект, а shared внутри layout-зоны.

В menu logic

sb_menu_get_site_record

sb_menu_upsert_site_record

sb_menu_next_menu_id

sb_menu_next_item_id

sb_menu_find_menu


Это локально для menu.


---

8. Как бы я разбил твои текущие файлы

api.php

Разбить обязательно.

Целевой результат:

api/index.php

api/site.php

api/page.php

api/block.php

api/menu.php

api/access.php

api/file.php

api/template.php

api/layout.php



---

editor.php

Оставить как entry-point страницы редактора, но вынести:

JS в assets/js/editor/*

повторяющиеся куски HTML в views/editor/*


В editor.php должно остаться:

подключение пролога

загрузка данных страницы

вывод контейнеров

подключение CSS/JS



---

layout.php

Аналогично:

страница + контейнеры + подключение ассетов

JS-логика наружу в assets/js/layout/*



---

public.php

Сейчас это толстый рендер. Его делить так:

src/PublicPage/PageTree.php

src/PublicPage/Breadcrumbs.php

src/PublicPage/BlockRenderer.php

src/PublicPage/PublicRenderer.php


public.php оставить только как публичную точку входа.


---

view.php

Скорее всего это административный/preview просмотр. Нужно вынести:

tree helpers

breadcrumbs

page lookup


Если пересекается с public.php, не копировать, а переиспользовать код из src/PublicPage.


---

download.php

Оставить отдельным entry point, но убрать дубли:

ACL

data path

доступ к site disk folder


Пусть использует:

Auth/Access.php

Repositories/SitesRepository.php

Disk/DiskStorage.php



---

9. Пошаговый безопасный план рефакторинга

Это важно делать без “большого взрыва”.

Этап 1. Создать каркас папок

Не переносить логику сразу. Просто создать:

/api

/src

/assets/js

/assets/css

/views



---

Этап 2. Вынести самую безопасную общую инфраструктуру

Сначала выносятся функции, которые ничего не меняют по бизнес-логике:

JsonStorage

Response

Access

Slug

DiskStorage


После этого старые файлы просто начинают подключать их.


---

Этап 3. Разрезать api.php

Сначала без переписывания логики:

скопировать action-блоки в отдельные файлы

оставить старую реализацию внутри этих файлов

в api/index.php сделать диспетчер


То есть сначала перенос, а не реинжиниринг.


---

Этап 4. Вынести repositories

После стабилизации API:

SitesRepository

PagesRepository

BlocksRepository

MenusRepository

и т.д.


И заменить прямые sb_read_* / sb_write_* на методы репозиториев.


---

Этап 5. Вынести Disk

Потом выделить весь Disk-код в один слой.


---

Этап 6. Разрезать editor.php и layout.php

Сначала вынести JS как есть, почти без изменений:

кусок в отдельный .js

подключить его на странице

проверить, что всё работает


Только потом резать на dialogs, render, actions.


---

Этап 7. Разрезать public.php

После стабилизации административной части.


---

Этап 8. Убрать дубли

Когда новые слои уже используются:

удалить дублированные helpers из public.php, download.php, view.php, api.php



---

10. Практические правила именования

Для PHP

классы: SiteService, PagesRepository, DiskStorage

методы: createSite, updatePageMeta, deleteBlock, syncFolderRights

не использовать абстрактные названия типа Manager, Helper, Processor, если можно назвать по смыслу


Для API handlers

api/site.php

api/page.php

api/layout.php


Для JS

blocks-render.js

block-actions.js

template-actions.js

dialogs/gallery.js


То есть файл должен отвечать на вопрос: “за что он отвечает”.


---

11. Минимальный вариант, если не хочется вводить много классов

Если хочешь совсем без лишнего ООП, можно сделать облегчённую схему:

/local/sitebuilder/
├── api/
│   ├── index.php
│   ├── site.php
│   ├── page.php
│   ├── block.php
│   └── ...
├── lib/
│   ├── json.php
│   ├── access.php
│   ├── disk.php
│   ├── site.php
│   ├── page.php
│   ├── block.php
│   ├── menu.php
│   ├── template.php
│   ├── layout.php
│   └── public.php
├── assets/js/
├── assets/css/
├── views/
└── *.php

Это тоже нормальный путь для Bitrix24-коробки, если не хочется делать src/Shared, src/Services, src/Repositories.

То есть:

минималистично: через lib/*.php

чуть чище и на вырост: через src/*


Для твоего проекта я бы выбрал средний вариант:

api/

src/Shared

src/Auth

src/Disk

src/Repositories

src/PublicPage

assets/js

views


А вот слишком много service-классов создавал бы только там, где реально есть смысл.


---

12. Что я бы сделал первым именно у тебя

Если прямо по текущему архиву, мой первый безопасный шаг был бы такой:

1. Создать api/index.php


2. Перенести action-блоки из api.php по файлам:

api/site.php

api/page.php

api/block.php

api/menu.php

api/template.php

api/access.php

api/file.php

api/layout.php



3. Вынести в src/Shared/JsonStorage.php повторяющиеся sb_read_json_file/sb_write_json_file


4. Вынести ACL в src/Auth/Access.php


5. Вынести Disk в src/Disk/DiskStorage.php


6. Только после этого начинать резать editor.php



Потому что главная боль сейчас — backend-монолит api.php.


---

Если хочешь, следующим сообщением я могу уже на основе твоего архива дать тебе готовое целевое дерево именно с раскладкой каждого текущего файла:
api.php → куда, public.php → куда, editor.php → на какие JS/PHP части, layout.php → на какие части.