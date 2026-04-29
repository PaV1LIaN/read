Отлично. Следующий этап — **привязать права диска к ролям сайта**, чтобы настройки блока не были единственным ограничением.

Сделаем так:

```text
VIEWER
- видеть файлы
- скачивать

EDITOR
- видеть файлы
- скачивать
- загружать
- создавать папки
- переименовывать

ADMIN / OWNER
- всё выше
- удалять
- управлять настройками
```

Важно: это надо делать **на сервере**, не только скрывать кнопки в интерфейсе. Иначе пользователь сможет дернуть `disk.api` руками.

Мне нужен текущий файл, где в дисковом компоненте формируется ответ `permissions`:

Скорее всего один из этих:

```text
/local/sitebuilder/components/disk/lib/DiskPermissionService.php
/local/sitebuilder/components/disk/lib/DiskPermissionResolver.php
/local/sitebuilder/components/disk/lib/DiskContext.php
/local/sitebuilder/components/disk/api.php
/local/sitebuilder/components/disk/api/index.php
```

И ещё файл, где обрабатываются действия диска:

```text
bootstrap
list
upload
createFolder
rename
delete
download
```

Пришли файл, в котором сейчас есть массив примерно такого вида:

```php
'permissions' => [
    'canView' => true,
    'canUpload' => true,
    'canCreateFolder' => true,
    'canRename' => true,
    'canDelete' => false,
    'canDownload' => true,
    'canManageAccess' => false,
    'canEditSettings' => false,
]
```

После этого я дам полный готовый файл, чтобы права диска брались из роли `sitebuilder.access`, а не просто из настроек блока.
