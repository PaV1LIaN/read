В текущем виде file.php работает только с общим Диском сайта:

file.list проверяет глобальный VIEWER;

file.upload и file.delete проверяют глобальный EDITOR;

запросы не содержат pageId;

запросы не содержат blockId;

сервер не может определить, к какой странице и к какому блоку Диска относится операция.


Поэтому этот файл пока не заменяем. Сначала нужно добавить права Диска в sitebuilder.page_access.

Следующий шаг: расширить таблицу прав

В PostgreSQL выполни:

ALTER TABLE sitebuilder.page_access
    ADD COLUMN IF NOT EXISTS can_disk_view BOOLEAN NOT NULL DEFAULT FALSE,
    ADD COLUMN IF NOT EXISTS can_disk_edit BOOLEAN NOT NULL DEFAULT FALSE;

Проверить структуру:

SELECT
    column_name,
    data_type,
    is_nullable,
    column_default
FROM information_schema.columns
WHERE table_schema = 'sitebuilder'
  AND table_name = 'page_access'
ORDER BY ordinal_position;

В таблице должны появиться:

can_disk_view
can_disk_edit

Итоговые права одной страницы

Запись будет содержать:

can_view          — просмотр страницы
can_edit          — изменение страницы и блоков
can_disk_view     — просмотр и скачивание файлов
can_disk_edit     — загрузка, создание папок, переименование, удаление
include_children  — наследование на дочерние страницы

Зависимости:

can_edit → can_view
can_disk_edit → can_disk_view

При этом can_disk_edit не должен автоматически давать can_edit: сотрудник сможет работать с документами, но не менять страницу.

Как глобальные роли будут работать с Диском

Глобальная роль	Диск

VIEWER	Просмотр и скачивание
EDITOR	Просмотр и изменение
ADMIN	Полный доступ
OWNER	Полный доступ
Нет глобальной роли	По can_disk_view/can_disk_edit


Новый контракт file.php

Каждый запрос к блоку Диска должен передавать:

siteId
pageId
blockId

Например:

api('file.list', {
    siteId: siteId,
    pageId: currentPageId,
    blockId: diskBlockId
});

На сервере должна проверяться вся цепочка:

сайт существует
↓
страница принадлежит сайту
↓
блок принадлежит странице
↓
тип блока = disk
↓
у пользователя есть disk.view или disk.edit
↓
операция разрешена настройками блока

Для удаления файла одновременно должны выполняться условия:

can_disk_edit = true
и
diskAllowDelete = true
и
файл находится внутри папки этого блока/сайта

Важное ограничение текущего файла

Сейчас используется:

$folder = sb_disk_ensure_site_folder($siteId);

То есть все операции идут в корень сайта. Параметр blockId не используется, а режим:

rootMode = site
rootMode = block

из настроек блока тоже не учитывается.

Нельзя сейчас просто заменить sb_require_editor() на canEditDisk(): пользователь с правом на одну страницу получил бы доступ ко всему Диску сайта.

После выполнения SQL следующим обновляем PageAccessRepository.php, добавляя canDiskView и canDiskEdit. Затем PageAccessService.php, и только после этого безопасно переписываем file.php.