Дальше идём по такому порядку.

1. Проверить синтаксис изменённых файлов

На сервере выполни:

php -l /srv/bx/docroot/local/sitebuilder/lib/PageAccessRepository.php
php -l /srv/bx/docroot/local/sitebuilder/api/handlers/page_access.php
php -l /srv/bx/docroot/local/sitebuilder/api/handlers/page.php
php -l /srv/bx/docroot/local/sitebuilder/api/handlers/site.php
php -l /srv/bx/docroot/local/sitebuilder/api/handlers/block.php
php -l /srv/bx/docroot/local/sitebuilder/editor.php

Везде должно быть:

No syntax errors detected

2. Проверить уникальный индекс

В PostgreSQL выполни:

CREATE UNIQUE INDEX IF NOT EXISTS uq_page_access_site_page_code
ON sitebuilder.page_access (
    site_id,
    page_id,
    access_code
);

Проверить существование:

SELECT
    indexname,
    indexdef
FROM pg_indexes
WHERE schemaname = 'sitebuilder'
  AND tablename = 'page_access';

3. Проверить права на блоки

Нужны тестовые пользователи либо временная смена ролей.

Глобальный VIEWER

Должен:

видеть все страницы;

получать block.list;

не создавать блок;

не изменять блок;

не удалять блок.


Глобальный EDITOR

Должен:

видеть все страницы;

создавать и изменять блоки на всех страницах.


Пользователь только с page.view

Должен:

видеть разрешённую страницу;

получать блоки разрешённой страницы;

получать 403 PAGE_VIEW_ACCESS_DENIED на другой странице;

получать 403 PAGE_EDIT_ACCESS_DENIED при попытке изменить блок.


Пользователь только с page.edit

Должен:

открыть редактор;

видеть разрешённую страницу и родителей navigationOnly;

создавать, изменять, перемещать и удалять блоки разрешённой страницы;

получать отказ на остальных страницах.


Проверка наследования

Выдать право родительской странице:

canView = true
canEdit = true
includeChildren = true

Пользователь должен получить редактирование всех дочерних страниц.

4. Следующий этап — права Диска

Сейчас page.edit защищает изменение самого блока Диска, например его заголовок и настройки. Но операции с файлами пока требуют отдельной проверки:

просмотр списка файлов;

скачивание;

загрузка;

создание папки;

переименование;

удаление.


Следующим пришли полный файл:

/local/sitebuilder/api/handlers/file.php

В нём добавим основу для:

disk.view
disk.edit

На первом этапе:

disk.view
→ список файлов, открытие и скачивание

disk.edit
→ disk.view + загрузка, папки, переименование и удаление

При этом настройки блока allowUpload, allowDelete, allowRename и остальные продолжат работать как дополнительное ограничение.