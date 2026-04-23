Нужно убрать вызов getDownloadUrl() — в твоей версии Bitrix этого метода у Bitrix\Disk\File нет.

Исправь normalizeFile() и getDownloadUrl() в адаптере.

Файл

/local/sitebuilder/components/disk/lib/DiskBitrixStorageAdapter.php

1. Замени normalizeFile() на это

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
        'downloadUrl' => $this->buildDownloadUrl((int)$file->getId()),
        'previewUrl' => '',
        'createdAt' => $this->normalizeDate($file->getCreateTime()),
        'updatedAt' => $this->normalizeDate($file->getUpdateTime()),
        'createdBy' => (int)$file->getCreatedBy(),
    ];
}

2. Замени getDownloadUrl() на это

public function getDownloadUrl(DiskContext $context, int $fileId): string
{
    $this->getFileById($fileId);
    return $this->buildDownloadUrl($fileId);
}

3. Добавь в класс новый метод

protected function buildDownloadUrl(int $fileId): string
{
    return '/bitrix/tools/disk/uf.php?attachedId=' . (int)$fileId . '&action=download&ncc=1';
}

Если этот URL в твоей установке не подойдет, тогда используем другой, но сначала попробуй его.


---

Почему это работает

Мы больше не вызываем несуществующий метод getDownloadUrl(), а сами строим ссылку на скачивание.


---

После этого

1. сохрани файл


2. Ctrl + F5


3. обнови страницу с блоком disk




---

Что ожидаю дальше

После этой правки:

список файлов должен начать отображаться

загрузка должна пройти до конца

кнопка “Открыть” у файла будет пытаться скачивать его по построенному URL


Если после этого файл загрузится, но скачивание не откроется, пришли response и я дам URL именно под твою сборку Bitrix Disk.