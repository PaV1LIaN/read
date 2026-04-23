Проблема точная: ты сейчас строишь ссылку как для attached object:

/bitrix/tools/disk/uf.php?attachedId=341&action=download&ncc=1

Но у тебя в disk хранится и выводится обычный Bitrix\Disk\File id, а не attachedId.
Поэтому Bitrix и отвечает:

{"status":"error","errors":[{"message":"Could not find attached object","code":0}]}


---

Что нужно исправить

Нужно перестать строить URL через attachedId и получать ссылку на скачивание для самого Disk File.

Файл

/local/sitebuilder/components/disk/lib/DiskBitrixStorageAdapter.php

1. Замени метод getDownloadUrl(...) на это

public function getDownloadUrl(DiskContext $context, int $fileId): string
{
    $file = $this->getFileById($fileId);
    return $this->buildDownloadUrl($file);
}

2. Замени normalizeFile() на это

protected function normalizeFile(DiskContext $context, File $file): array
{
    $name = (string)$file->getName();
    $extension = (string)$file->getExtension();
    $mimeType = $this->detectMimeTypeByExtension($extension);

    return [
        'id' => (int)$file->getId(),
        'entityType' => 'file',
        'name' => $name,
        'originalName' => $name,
        'extension' => $extension,
        'mimeType' => $mimeType,
        'size' => (int)$file->getSize(),
        'downloadUrl' => $this->buildDownloadUrl($file),
        'previewUrl' => '',
        'createdAt' => $this->normalizeDate($file->getCreateTime()),
        'updatedAt' => $this->normalizeDate($file->getUpdateTime()),
        'createdBy' => (int)$file->getCreatedBy(),
    ];
}

3. Замени buildDownloadUrl(...) на это

protected function buildDownloadUrl(File $file): string
{
    $driver = \Bitrix\Disk\Driver::getInstance();
    $urlManager = $driver->getUrlManager();

    if (is_object($urlManager) && method_exists($urlManager, 'getUrlForDownloadFile')) {
        return (string)$urlManager->getUrlForDownloadFile($file, true);
    }

    // fallback, если в твоей версии метода нет
    return '/bitrix/tools/disk/downloadFile/' . (int)$file->getId() . '/?ncc=1';
}


---

Почему это должно помочь

Сейчас у тебя:

341 — это disk file id

uf.php?attachedId=341 ищет attached object

attached object с таким id нет


После правки будет использоваться URL именно для Bitrix\Disk\File.


---

Что сделать после правки

1. сохранить DiskBitrixStorageAdapter.php


2. Ctrl + F5


3. обновить страницу с disk


4. попробовать открыть файл снова




---

Если после этого не скачает

Тогда нужно будет посмотреть, какой URL реально сгенерировался.

В консоли браузера выполни:

document.querySelector('[data-entity-type="file"]')?.getAttribute('data-download-url')

И пришли результат. Тогда я скажу, подходит ли этот URL под твою версию Bitrix Disk.