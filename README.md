Ошибка точная: в normalizeFile() метод buildDiskOpenUrl() вызывается в старом порядке аргументов.

Сейчас у тебя где-то так:

$this->buildDiskOpenUrl($file)

или вообще перепутано:

$this->buildDiskOpenUrl($file, $context)

А метод уже объявлен как:

buildDiskOpenUrl(DiskContext $context, File $file)

Что исправить

Файл:

/local/sitebuilder/components/disk/lib/DiskBitrixStorageAdapter.php

Замени normalizeFile() целиком на этот вариант

protected function normalizeFile(DiskContext $context, File $file): array
{
    $name = (string)$file->getName();
    $extension = (string)$file->getExtension();
    $mimeType = $this->detectMimeTypeByExtension($extension);

    $isOffice = $this->isOfficeDocument($extension);

    return [
        'id' => (int)$file->getId(),
        'entityType' => 'file',
        'name' => $name,
        'originalName' => $name,
        'extension' => $extension,
        'mimeType' => $mimeType,
        'size' => (int)$file->getSize(),
        'downloadUrl' => $this->buildDownloadUrl($file),
        'previewUrl' => $isOffice
            ? $this->buildDiskOpenUrl($context, $file)
            : $this->buildPreviewUrl($context, $file),
        'editUrl' => $isOffice
            ? $this->buildDiskEditUrl($context, $file)
            : '',
        'previewMode' => $isOffice ? 'office' : 'browser',
        'canEdit' => $isOffice,
        'createdAt' => $this->normalizeDate($file->getCreateTime()),
        'updatedAt' => $this->normalizeDate($file->getUpdateTime()),
        'createdBy' => (int)$file->getCreatedBy(),
    ];
}

И проверь, что методы ниже объявлены именно так

protected function buildDiskOpenUrl(DiskContext $context, File $file): string
{
    return '/local/sitebuilder/components/disk/open.php'
        . '?siteId=' . (int)$context->siteId
        . '&pageId=' . (int)$context->pageId
        . '&blockId=' . (int)$context->blockId
        . '&fileId=' . (int)$file->getId()
        . '&mode=view';
}

protected function buildDiskEditUrl(DiskContext $context, File $file): string
{
    return '/local/sitebuilder/components/disk/open.php'
        . '?siteId=' . (int)$context->siteId
        . '&pageId=' . (int)$context->pageId
        . '&blockId=' . (int)$context->blockId
        . '&fileId=' . (int)$file->getId()
        . '&mode=edit';
}

Почему упало

PHP пишет, что в первый аргумент метода попал Bitrix\Disk\File, значит вызов был не такой, как сигнатура метода.

После этого

Сохрани файл и сделай Ctrl + F5.

Если после этого снова будет ошибка, пришли текущие методы:

normalizeFile()

buildDiskOpenUrl()

buildDiskEditUrl()