У тебя в DiskBitrixStorageAdapter.php метод isOfficeDocument() уже был, и ты добавил его второй раз.

Нужно оставить только один.

Что сделать

Открой файл:

/local/sitebuilder/components/disk/lib/DiskBitrixStorageAdapter.php

Найди оба метода:

protected function isOfficeDocument(string $extension): bool

и удали один из них.

Оставь только такой вариант:

protected function isOfficeDocument(string $extension): bool
{
    $extension = mb_strtolower(trim($extension));

    return in_array($extension, [
        'doc', 'docx',
        'xls', 'xlsx',
        'ppt', 'pptx',
        'odt', 'ods', 'odp',
        'rtf', 'csv'
    ], true);
}

Что еще проверить сразу

Скорее всего ты так же мог продублировать и другие методы. Проверь, чтобы в классе был только один экземпляр каждого из этих методов:

isOfficeDocument

buildViewerSrc

buildPreviewUrl

buildDownloadUrl

detectMimeTypeByExtension


Быстрый способ найти дубли

По SSH выполни:

grep -n "function isOfficeDocument" /srv/bx/docroot/local/sitebuilder/components/disk/lib/DiskBitrixStorageAdapter.php
grep -n "function buildViewerSrc" /srv/bx/docroot/local/sitebuilder/components/disk/lib/DiskBitrixStorageAdapter.php
grep -n "function buildPreviewUrl" /srv/bx/docroot/local/sitebuilder/components/disk/lib/DiskBitrixStorageAdapter.php
grep -n "function buildDownloadUrl" /srv/bx/docroot/local/sitebuilder/components/disk/lib/DiskBitrixStorageAdapter.php

Если по какому-то имени показывает 2 строки — значит этот метод тоже задублирован.

После удаления дубля обнови страницу. Если хочешь, пришли мне текущий DiskBitrixStorageAdapter.php, и я соберу тебе его в один чистый готовый файл.