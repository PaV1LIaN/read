Значит disk.viewer.document-item у тебя не подгрузился, и из-за этого viewer недоступен.

Нужно подключать не только ui.viewer, а еще disk viewer-расширения, и делать это до ShowHead().

Что вставить в public_page.php

Файл:

/local/sitebuilder/views/layout/public_page.php

В самый верх файла, до <!doctype html>, вставь:

<?php
global $APPLICATION;

if ($pageHasDiskBlock) {
    \CJSCore::Init(['ui.viewer']);

    if (\Bitrix\Main\Loader::includeModule('disk')) {
        \Bitrix\Main\UI\Extension::load([
            'disk.viewer',
            'disk.document',
            'disk.viewer.document-item',
        ]);
    }
}
?>


---

Что должно быть в <head>

Проверь, что внутри <head> есть:

<?php $APPLICATION->ShowHead(); ?>

Пример:

<head>
    <meta charset="UTF-8">
    <?php $APPLICATION->ShowHead(); ?>
    <title>...</title>


---

Почему сейчас не работает

У тебя сейчас в консоли:

typeof BX.viewElementBind // undefined
BX.Disk.Viewer // no disk viewer

Это значит, что страница вообще не получила JS Bitrix Disk Viewer.
Пока это не загрузится, Открыть и Редактировать для docx будут сваливаться в скачивание.


---

После правки

Сделай Ctrl + F5 и проверь в консоли:

typeof BX.viewElementBind

и

typeof BX.Disk !== 'undefined' && BX.Disk.Viewer && BX.Disk.Viewer.Actions ? Object.keys(BX.Disk.Viewer.Actions) : 'no disk viewer'

Если снова будет undefined и no disk viewer, тогда нужно проверить, грузятся ли эти расширения вообще. Для этого открой Network и посмотри, есть ли запросы к скриптам disk.viewer / disk.document.

Если хочешь, пришли мне верхнюю часть public_page.php целиком, я скажу, куда именно вставить код без гадания.