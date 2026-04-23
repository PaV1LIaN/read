Причина теперь точная:

Название содержит недопустимые символы

У тебя ломает имя папки.
Скорее всего из-за такого формата:

'Сайт: ' . $siteName
'Блок: ' . $blockTitle

Для Bitrix Disk символ : часто недопустим.
Нужно санитизировать имена папок перед созданием.


---

Что исправить

1. Создай helper для безопасного имени папки

Файл:

/local/sitebuilder/components/disk/lib/DiskNameSanitizer.php

<?php

class DiskNameSanitizer
{
    public static function sanitizeFolderName(string $name, string $fallback = 'Новая папка'): string
    {
        $name = trim($name);

        // Убираем явно проблемные символы для Disk / файловых имен
        $name = preg_replace('/[\\\\\\/\\:\\*\\?"<>\\|]+/u', ' ', $name);

        // Схлопываем пробелы
        $name = preg_replace('/\\s+/u', ' ', $name);

        $name = trim($name, " \t\n\r\0\x0B.-_");

        if ($name === '') {
            $name = $fallback;
        }

        // Ограничим длину
        if (mb_strlen($name) > 100) {
            $name = mb_substr($name, 0, 100);
            $name = rtrim($name, " \t\n\r\0\x0B.-_");
        }

        if ($name === '') {
            $name = $fallback;
        }

        return $name;
    }
}


---

2. Подключи helper в bootstrap.php

Файл:

/local/sitebuilder/components/disk/bootstrap.php

Добавь:

require_once __DIR__ . '/lib/DiskNameSanitizer.php';


---

3. Исправь SiteDiskInitializer.php

Файл:

/local/sitebuilder/components/disk/lib/SiteDiskInitializer.php

Найди:

$folderName = $siteName !== ''
    ? ('Сайт: ' . $siteName)
    : ('Сайт: ' . (string)$site['name']);

И замени на:

$folderBaseName = $siteName !== ''
    ? ('Сайт ' . $siteName)
    : ('Сайт ' . (string)$site['name']);

$folderName = DiskNameSanitizer::sanitizeFolderName($folderBaseName, 'Сайт');


---

4. Исправь BlockDiskInitializer.php

Файл:

/local/sitebuilder/components/disk/lib/BlockDiskInitializer.php

Найди:

$folderName = $blockTitle !== ''
    ? ('Блок: ' . $blockTitle)
    : ('Блок #' . $blockId);

И замени на:

$folderBaseName = $blockTitle !== ''
    ? ('Блок ' . $blockTitle)
    : ('Блок ' . $blockId);

$folderName = DiskNameSanitizer::sanitizeFolderName($folderBaseName, 'Блок');


---

5. Исправь DiskBitrixStorageAdapter.php для ручного создания папок

Чтобы пользователь тоже не мог создать папку с запрещенными символами.

Файл:

/local/sitebuilder/components/disk/lib/DiskBitrixStorageAdapter.php

В методе createFolder(...) найди:

$createdFolder = $parentFolder->addSubFolder([
    'NAME' => $name,
    'CREATED_BY' => $context->currentUserId,
], [], true);

И замени на:

$safeName = DiskNameSanitizer::sanitizeFolderName($name, 'Новая папка');

$createdFolder = $parentFolder->addSubFolder([
    'NAME' => $safeName,
    'CREATED_BY' => $context->currentUserId,
], [], true);


---

Что делать после этого

1. сохранить 3 файла


2. обновить страницу


3. снова открыть публичную страницу с disk


4. проверить bootstrap response




---

Что должно быть дальше

Теперь ошибка про недопустимые символы должна исчезнуть.
Ожидаемый следующий ответ:

{
  "ok": true,
  "data": {
    "rootFolderId": ...,
    "currentFolderId": ...,
    "rootSource": "site"
  }
}


---

Почему это надо сделать обязательно

Не только для авто-корня сайта, но и вообще для всех папок:

название сайта может содержать : / \ ? *

заголовок блока тоже

пользователь руками тоже может ввести плохое имя


Так что sanitize нужен как постоянный слой.

Если после этого вылезет следующая ошибка, пришли новый bootstrap response.