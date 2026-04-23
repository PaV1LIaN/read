Дальше нужно сделать 4 вещи, чтобы блок disk реально заработал на сайте, а не только отрисовался как контейнер.

1. Подключить disk.bootstrap на сервере

Ты уже вставил публичный рендер блока. Теперь нужно, чтобы API disk умел стартовать в контексте сайта.

Проверь, что у тебя уже есть:

/local/sitebuilder/components/disk/actions/bootstrap.php

в /local/sitebuilder/components/disk/api.php есть:


case 'bootstrap':
    require __DIR__ . '/actions/bootstrap.php';
    break;

Если этого еще нет — добавить.


---

2. Обновить script.js, чтобы он стартовал через bootstrap

Нужно, чтобы фронт блока не пытался жить только на data-initial-state, а сразу дергал сервер и получал:

реальные настройки из block.props

права пользователя

rootFolderId

currentFolderId


В /local/sitebuilder/components/disk/script.js найди:

DiskComponent.prototype.init = async function () {

и замени на:

DiskComponent.prototype.init = async function () {
  this.bindStaticEvents();

  try {
    var payload = this.getBasePayload();
    payload.sessid = this.getSessid();

    var res = await this.api('bootstrap', payload);
    if (!res.ok) {
      throw new Error(res.message || res.error || 'BOOTSTRAP_ERROR');
    }

    var data = res.data || {};

    this.state.siteId = Number(data.siteId || this.state.siteId || 0);
    this.state.pageId = Number(data.pageId || this.state.pageId || 0);
    this.state.blockId = Number(data.blockId || this.state.blockId || 0);
    this.state.settings = data.settings || {};
    this.state.permissions = data.permissions || {};
    this.state.rootFolderId = data.rootFolderId || null;
    this.state.currentFolderId = data.currentFolderId || null;
    this.state.viewMode = (this.state.settings && this.state.settings.viewMode) || 'table';

    this.applyInitialViewMode();

    if (!this.state.permissions.canView) {
      this.renderState('no-access');
      return;
    }

    if (!this.state.rootFolderId) {
      this.renderState('no-root');
      return;
    }

    await this.loadFolder(this.state.rootFolderId);
  } catch (e) {
    console.error(e);
    this.renderState('error');
  }
};


---

3. Проверить storage-интеграцию блока

Теперь нужно убедиться, что disk берет настройки не из старой таблицы, а из sitebuilder.block.props_json.

То есть у тебя уже должны быть заменены:

DiskSitebuilderBridge.php

SiteRepository.php

BlockRepository.php

DiskSettingsRepository.php

DiskRootResolver.php

BlockDiskInitializer.php

SiteAccessRepository.php


Если что-то из этого еще не заменено, bootstrap может стартовать, но читать не тот источник.


---

4. Проверить сценарий запуска на странице

Теперь можно тестировать по шагам.

Сценарий A. Блок открыт впервые

Открой публичную страницу, где есть disk.

Ожидаемое поведение:

блок рендерится

идет bootstrap

создается корень сайта, если его нет

либо создается папка блока, если rootMode = block

грузится содержимое


Сценарий B. У пользователя нет прав

Поставь пользователю роль VIEWER и проверь:

блок открывается

доступ только на просмотр

без загрузки и удаления


Сценарий C. Настройки блока

В редакторе создай или отредактируй disk блок и в props задай:

rootMode = site

потом rootMode = block


И проверь, что блок показывает нужную файловую зону.


---

Что проверить в первую очередь прямо сейчас

После того как ты вставил публичный рендер, я бы делал так:

1.

Открой страницу с блоком disk

2.

Открой DevTools → Network

3.

Посмотри, есть ли запрос:

/local/sitebuilder/components/disk/api.php?action=bootstrap

4.

Если запрос есть:

смотри его response


Ожидаемо там должно быть что-то вроде:

{
  "ok": true,
  "data": {
    "siteId": 1,
    "pageId": 10,
    "blockId": 55,
    "settings": {...},
    "permissions": {...},
    "rootFolderId": 123,
    "currentFolderId": 123,
    "rootSource": "site"
  }
}


---

Если bootstrap не вызывается

Значит проблема еще на фронте:

не подключился script.js

не сработал DOMContentLoaded

класс .sb-disk не найден

script.js еще старый



---

Если bootstrap вызывается, но падает

Тогда пришли response этого запроса. Это уже сразу покажет, что сломано:

context

права

root

props

block type

Bitrix Disk



---

Если хочешь идти без диагностики, а сразу дальше

Следующий логичный шаг после запуска на публичной странице — сделать человеческую настройку блока disk в editor.php, чтобы не редактировать его через JSON.

То есть:

поле “Заголовок”

выбор rootMode

чекбоксы прав

viewMode

maxFileSize

allowedExtensions


Это как раз следующий удобный этап.