Нужно исправить вызов uploadFile() в адаптере: в твоей версии Bitrix третий аргумент — это массив прав, а не true.

Открой файл:

/local/sitebuilder/components/disk/lib/DiskBitrixStorageAdapter.php

Найди в методе uploadFiles(...) этот кусок:

$createdFile = $folder->uploadFile(
    [
        'NAME' => $safeName,
        'TMP_NAME' => $file['tmp_name'],
        'TYPE' => $file['type'] ?? '',
        'SIZE' => (int)($file['size'] ?? 0),
        'CREATED_BY' => $context->currentUserId,
    ],
    [],
    true
);

И замени на:

$createdFile = $folder->uploadFile(
    [
        'NAME' => $safeName,
        'TMP_NAME' => $file['tmp_name'],
        'TYPE' => $file['type'] ?? '',
        'SIZE' => (int)($file['size'] ?? 0),
        'CREATED_BY' => $context->currentUserId,
    ],
    [],
    []
);

Если хочешь сразу без лишних полей, можно так:

$createdFile = $folder->uploadFile(
    [
        'NAME' => $safeName,
        'TMP_NAME' => $file['tmp_name'],
        'TYPE' => $file['type'] ?? '',
        'SIZE' => (int)($file['size'] ?? 0),
        'CREATED_BY' => $context->currentUserId,
    ],
    [],
    []
);

Полный метод лучше сделать таким:

public function uploadFiles(DiskContext $context, int $folderId, array $files, array $settings = []): array
{
    $folder = $this->getFolderById($folderId);
    $uploaded = [];

    foreach ($files as $file) {
        $safeName = DiskNameSanitizer::sanitizeFolderName((string)$file['name'], 'file');

        $createdFile = $folder->uploadFile(
            [
                'NAME' => $safeName,
                'TMP_NAME' => $file['tmp_name'],
                'TYPE' => $file['type'] ?? '',
                'SIZE' => (int)($file['size'] ?? 0),
                'CREATED_BY' => $context->currentUserId,
            ],
            [],
            []
        );

        if ($createdFile instanceof File) {
            $uploaded[] = $this->normalizeFile($context, $createdFile);
        } else {
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
    }

    return $uploaded;
}

После этого:

1. сохрани файл


2. сделай Ctrl + F5


3. попробуй загрузить файл снова



Если появится следующая ошибка, пришли response api.php?action=upload.