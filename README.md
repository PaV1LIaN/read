Теперь причина уже другая и она понятна:

когда ты начал рендерить полный шаблон компонента через template.php, ты снова потерял data-sessid.

То есть:

bootstrap у тебя работает, потому что, скорее всего, первый рендер еще брал sessid

а вот дальнейшие действия createFolder, upload, saveSettings берут sessid из корневого .sb-disk

и в template.php этого атрибута сейчас нет



---

Что исправить

Файл

/local/sitebuilder/components/disk/template.php

Найди начало контейнера, сейчас оно у тебя примерно такое:

<div class="sb-disk"
     id="sb-disk-<?= (int)$arResult['BLOCK_ID'] ?>"
     data-site-id="<?= (int)$arResult['SITE_ID'] ?>"
     data-page-id="<?= (int)$arResult['PAGE_ID'] ?>"
     data-block-id="<?= (int)$arResult['BLOCK_ID'] ?>"
     data-initial-state="<?= $initialStateJson ?>">

И замени на:

<div class="sb-disk"
     id="sb-disk-<?= (int)$arResult['BLOCK_ID'] ?>"
     data-site-id="<?= (int)$arResult['SITE_ID'] ?>"
     data-page-id="<?= (int)$arResult['PAGE_ID'] ?>"
     data-block-id="<?= (int)$arResult['BLOCK_ID'] ?>"
     data-sessid="<?= htmlspecialchars(bitrix_sessid(), ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>"
     data-initial-state="<?= $initialStateJson ?>">


---

Почему это чинит

script.js у тебя уже исправлен и берет sessid так:

1. пробует BX.bitrix_sessid()


2. если там пусто — берет data-sessid у .sb-disk



Но в полном шаблоне компонента этот data-sessid сейчас не передается, поэтому для createFolder и других POST-действий снова уходит пустой sessid.


---

После этого

1. сохрани template.php


2. сделай Ctrl + F5


3. снова попробуй создать папку




---

Что еще советую проверить сразу

В консоли браузера выполни:

document.querySelector('.sb-disk')?.getAttribute('data-sessid')

Должна вернуться строка sessid, не null.

Если после этого будет новая ошибка, пришли response createFolder.