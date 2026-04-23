Теперь ошибка уже предметная:

Required params: { NAME }

Значит в uploadFile() твоя версия Bitrix ожидает другой формат аргументов, и поле NAME она не видит в том массиве, который мы передаем.

Для твоей сборки safest way — передавать upload как обычный $_FILES-подобный массив первым аргументом, а не смешанный набор полей.

Что заменить

Файл:

/local/sitebuilder/components/disk/lib/DiskBitrixStorageAdapter.php

Замени метод uploadFiles(...) целиком на этот:

public function uploadFiles(DiskContext $context, int $folderId, array $files, array $settings = []): array
{
    $folder = $this->getFolderById($folderId);
    $uploaded = [];

    foreach ($files as $file) {
        $safeName = DiskNameSanitizer::sanitizeFolderName((string)($file['name'] ?? ''), 'file');

        $fileArray = [
            'name' => $safeName,
            'type' => (string)($file['type'] ?? ''),
            'tmp_name' => (string)($file['tmp_name'] ?? ''),
            'error' => (int)($file['error'] ?? 0),
            'size' => (int)($file['size'] ?? 0),
        ];

        $createdFile = $folder->uploadFile(
            $fileArray,
            [
                'NAME' => $safeName,
                'CREATED_BY' => $context->currentUserId,
            ],
            []
        );

        if ($createdFile instanceof File) {
            $uploaded[] = $this->normalizeFile($context, $createdFile);
            continue;
        }

        $errors = [];

        if (method_exists($folder, 'getErrors')) {
            foreach ((array)$folder->getErrors() as $error) {
                if (is_object($error) && method_exists($error, 'getMessage')) {
                    $errors[] = $error->getMessage();
                } else {
                    $errors[] = (string)$error;
                }
            }
        }

        throw new RuntimeException(
            'DISK_UPLOAD_FILE_ERROR' . (!empty($errors) ? ': ' . implode(' | ', $errors) : '')
        );
    }

    return $uploaded;
}

Почему так

В твоей версии Folder::uploadFile() похоже ждет:

1. массив файла в формате $_FILES


2. массив данных объекта с NAME


3. массив прав



А не один объединенный массив.

После этого

Сохрани файл, сделай Ctrl + F5 и попробуй загрузку снова.

Если будет следующая ошибка, пришли response api.php?action=upload.