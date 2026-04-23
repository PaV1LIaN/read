Причина найдена точно:

typeof BX.viewElementBind === 'undefined'

BX.Disk.Viewer вообще отсутствует


Значит на твоей кастомной странице не загружены js-расширения Disk Viewer, поэтому:

Открыть у office-файлов не инициализируется как viewer

Редактировать не может вызвать runActionDefaultEdit

код уходит в fallback, и файл просто скачивается


То есть проблема уже не в script.js, а в подключении битриксовых ассетов.

Что нужно сделать

1. В public_page.php до ShowHead() загрузить нужные расширения

Файл:

/local/sitebuilder/views/layout/public_page.php

В самый верх, до <!doctype html>, добавь:

<?php
global $APPLICATION;

if ($pageHasDiskBlock) {
    \CJSCore::Init(['ui.viewer']);
    \Bitrix\Main\UI\Extension::load([
        'disk.viewer.document-item',
    ]);
}
?>

Если use Bitrix\Main\UI\Extension; у тебя не подключен, можно так и оставить с полным namespace.


---

2. В <head> должен быть ShowHead()

Проверь, что у тебя есть:

<?php $APPLICATION->ShowHead(); ?>

внутри <head>.

Пример:

<head>
    <meta charset="UTF-8">
    <?php $APPLICATION->ShowHead(); ?>
    <title>...</title>


---

3. После этого в консоли должно стать так

Проверь:

typeof BX.viewElementBind

и

typeof BX.Disk !== 'undefined' && BX.Disk.Viewer && BX.Disk.Viewer.Actions ? Object.keys(BX.Disk.Viewer.Actions) : 'no disk viewer'

Ожидаемо:

первое перестанет быть undefined

второе перестанет быть no disk viewer



---

Почему сейчас скачивается файл

Потому что script.js делает fallback на previewUrl, а у office-файлов это:

/disk/downloadFile/349/?&ncc=1&filename=123.docx

То есть без viewer Bitrix это просто скачивание.


---

Что делать с script.js сейчас

Пока BX.Disk.Viewer не загружен, трогать script.js дальше бессмысленно.

Сначала добейся, чтобы в консоли появились:

BX.viewElementBind

BX.Disk.Viewer.Actions


После этого уже и Открыть, и Редактировать начнут работать как задумано.

Если после добавления Extension::load(['disk.viewer.document-item']) в консоли все еще будет no disk viewer, пришли кусок public_page.php с началом файла и <head>.