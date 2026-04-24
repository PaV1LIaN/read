Да, тогда следующий шаг делаем так: добавляем в интерфейс сайта блок “Группа Битрикс24 и права”.

Сначала лучше сделать маленькую доработку в API-данных сайта.

1. Добавь ссылку на группу в storage_db.php

В функции:

function sb_map_site_row(array $row): array

после:

'bitrixGroupCreatedAt' => !empty($row['bitrix_group_created_at']) ? (string)$row['bitrix_group_created_at'] : '',

добавь:

'bitrixGroupUrl' => !empty($row['bitrix_group_id'])
    ? '/workgroups/group/' . (int)$row['bitrix_group_id'] . '/'
    : '',

Получится такой кусок:

'bitrixGroupId' => !empty($row['bitrix_group_id']) ? (int)$row['bitrix_group_id'] : 0,
'bitrixGroupCreatedBy' => !empty($row['bitrix_group_created_by']) ? (int)$row['bitrix_group_created_by'] : 0,
'bitrixGroupCreatedAt' => !empty($row['bitrix_group_created_at']) ? (string)$row['bitrix_group_created_at'] : '',
'bitrixGroupUrl' => !empty($row['bitrix_group_id'])
    ? '/workgroups/group/' . (int)$row['bitrix_group_id'] . '/'
    : '',

После этого site.get будет возвращать не только ID, но и готовую ссылку.


---

2. Проверка

В консоли снова выполни:

fetch('/local/sitebuilder/api.php', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
  },
  body: new URLSearchParams({
    action: 'site.get',
    siteId: '11',
    sessid: BX.bitrix_sessid()
  }),
  credentials: 'same-origin'
})
.then(r => r.json())
.then(console.log);

В ответе должно появиться:

"bitrixGroupUrl": "/workgroups/group/2/"


---

3. Дальше нужен файл интерфейса

Теперь надо вставить блок с кнопкой в тот файл, где у тебя отображаются настройки/карточка сайта.

Скорее всего это один из файлов:

/local/sitebuilder/index.php
/local/sitebuilder/editor.php

или JS-файл, который рендерит список сайтов.

Пришли файл, где сейчас отображается карточка сайта или настройки сайта, и я вставлю туда готовый блок:

Группа Битрикс24
ID группы: 2
[Открыть группу]
[Синхронизировать права]

Плюс обработчик, который будет дергать:

action=site.syncAccess

и показывать результат:

Создано: 0
Обновлено: 0
Удалено: 0
Без изменений: 1