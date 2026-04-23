У тебя в этой версии Bitrix у Bitrix\Disk\File нет метода getMimeType().

Нужно поправить normalizeFile().

Файл

/local/sitebuilder/components/disk/lib/DiskBitrixStorageAdapter.php

Найди метод:

protected function normalizeFile(DiskContext $context, File $file): array

И замени целиком на это:

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
        'downloadUrl' => (string)$file->getDownloadUrl(),
        'previewUrl' => '',
        'createdAt' => $this->normalizeDate($file->getCreateTime()),
        'updatedAt' => $this->normalizeDate($file->getUpdateTime()),
        'createdBy' => (int)$file->getCreatedBy(),
    ];
}

И ниже в этом же классе добавь новый метод:

protected function detectMimeTypeByExtension(string $extension): string
{
    $extension = mb_strtolower(trim($extension));

    $map = [
        'pdf' => 'application/pdf',
        'doc' => 'application/msword',
        'docx' => 'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
        'xls' => 'application/vnd.ms-excel',
        'xlsx' => 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
        'ppt' => 'application/vnd.ms-powerpoint',
        'pptx' => 'application/vnd.openxmlformats-officedocument.presentationml.presentation',
        'txt' => 'text/plain',
        'csv' => 'text/csv',
        'json' => 'application/json',
        'xml' => 'application/xml',
        'zip' => 'application/zip',
        'rar' => 'application/vnd.rar',
        '7z' => 'application/x-7z-compressed',
        'jpg' => 'image/jpeg',
        'jpeg' => 'image/jpeg',
        'png' => 'image/png',
        'gif' => 'image/gif',
        'webp' => 'image/webp',
        'svg' => 'image/svg+xml',
        'mp4' => 'video/mp4',
        'mov' => 'video/quicktime',
        'avi' => 'video/x-msvideo',
        'mp3' => 'audio/mpeg',
        'wav' => 'audio/wav',
    ];

    return $map[$extension] ?? 'application/octet-stream';
}

Почему это чинит

Мы больше не вызываем несуществующий метод getMimeType(), а определяем MIME по расширению файла.

После этого

1. сохрани файл


2. сделай Ctrl + F5


3. попробуй загрузку снова



Если вылезет следующая ошибка, пришли response api.php?action=upload.