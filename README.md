Отлично, теперь мы уже дошли до настоящей проблемы: роутер работает, Disk-папка находится, но именно загрузка файла не подходит под сигнатуру твоей версии Bitrix Disk.

Ошибка:

{
  "ok": false,
  "error": "DISK_ERROR",
  "message": "Cannot upload file to Disk folder"
}

значит, что ломается функция sb_disk_upload_file_to_folder() из lib/disk.php.

На коробках Bitrix это частая история: uploadFile() в разных версиях принимает аргументы по-разному.


---

Что сделать

Нужно заменить только функцию sb_disk_upload_file_to_folder() в файле:

/local/sitebuilder/lib/disk.php

Найди текущую функцию:

function sb_disk_upload_file_to_folder(Folder $folder, array $fileArray): File

и полностью замени её на эту:

if (!function_exists('sb_disk_upload_file_to_folder')) {
    function sb_disk_upload_file_to_folder(Folder $folder, array $fileArray): File
    {
        $securityContext = sb_disk_security_context();

        $name = (string)($fileArray['name'] ?? 'file');
        $tmpName = (string)($fileArray['tmp_name'] ?? '');
        $type = (string)($fileArray['type'] ?? 'application/octet-stream');
        $size = (int)($fileArray['size'] ?? 0);

        if ($tmpName === '' || !file_exists($tmpName)) {
            throw new RuntimeException('Uploaded tmp file not found');
        }

        $uploadFile = [
            'name' => $name,
            'tmp_name' => $tmpName,
            'type' => $type,
            'size' => $size,
            'error' => 0,
        ];

        $fields = [
            'NAME' => $name,
            'CREATED_BY' => (function () {
                global $USER;
                return is_object($USER) ? (int)$USER->GetID() : 0;
            })(),
        ];

        $attemptErrors = [];

        try {
            $created = $folder->uploadFile($uploadFile, $fields, [], true);
            if ($created instanceof File) {
                return $created;
            }
            if (is_array($created) && isset($created['OBJECT']) && $created['OBJECT'] instanceof File) {
                return $created['OBJECT'];
            }
        } catch (Throwable $e) {
            $attemptErrors[] = 'uploadFile(file, fields, [], true): ' . $e->getMessage();
        }

        try {
            $created = $folder->uploadFile($uploadFile, $fields, []);
            if ($created instanceof File) {
                return $created;
            }
            if (is_array($created) && isset($created['OBJECT']) && $created['OBJECT'] instanceof File) {
                return $created['OBJECT'];
            }
        } catch (Throwable $e) {
            $attemptErrors[] = 'uploadFile(file, fields, []): ' . $e->getMessage();
        }

        try {
            $created = $folder->uploadFile($uploadFile, $fields);
            if ($created instanceof File) {
                return $created;
            }
            if (is_array($created) && isset($created['OBJECT']) && $created['OBJECT'] instanceof File) {
                return $created['OBJECT'];
            }
        } catch (Throwable $e) {
            $attemptErrors[] = 'uploadFile(file, fields): ' . $e->getMessage();
        }

        if ($securityContext) {
            try {
                $created = $folder->uploadFile($uploadFile, $fields, $securityContext);
                if ($created instanceof File) {
                    return $created;
                }
                if (is_array($created) && isset($created['OBJECT']) && $created['OBJECT'] instanceof File) {
                    return $created['OBJECT'];
                }
            } catch (Throwable $e) {
                $attemptErrors[] = 'uploadFile(file, fields, securityContext): ' . $e->getMessage();
            }

            try {
                $created = $folder->uploadFile($uploadFile, $securityContext);
                if ($created instanceof File) {
                    return $created;
                }
                if (is_array($created) && isset($created['OBJECT']) && $created['OBJECT'] instanceof File) {
                    return $created['OBJECT'];
                }
            } catch (Throwable $e) {
                $attemptErrors[] = 'uploadFile(file, securityContext): ' . $e->getMessage();
            }
        }

        throw new RuntimeException('Cannot upload file to Disk folder. Attempts: ' . implode(' | ', $attemptErrors));
    }
}


---

Почему это должно помочь

Раньше у нас были слишком слабые варианты вызова.
Теперь функция пробует несколько сигнатур подряд:

uploadFile($file, $fields, [], true)

uploadFile($file, $fields, [])

uploadFile($file, $fields)

uploadFile($file, $fields, $securityContext)

uploadFile($file, $securityContext)


Это как раз покрывает основные несовместимости коробочных версий Bitrix.


---

После замены

1. Сбрось opcache


2. Снова попробуй загрузить файл в files.php




---

Если снова будет ошибка

Тогда пришли новый текст полностью, потому что теперь он должен стать гораздо полезнее, например такого вида:

{
  "ok": false,
  "error": "DISK_ERROR",
  "message": "Cannot upload file to Disk folder. Attempts: uploadFile(file, fields, [], true): ... | uploadFile(file, fields): ..."
}

И по этой новой ошибке я уже точечно подгоню функцию именно под твою сигнатуру Disk API.