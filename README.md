Сейчас 404 почти наверняка на одном из URL, которые я дал для office-файлов:

previewUrl

editUrl


Именно они у тебя сейчас, скорее всего, указывают на несуществующий путь:

/bitrix/components/bitrix/disk.file.view/templates/.default/show_file.php

Для браузера это обычно непубличный путь шаблона, поэтому 404 — ожидаемо.

Что сделать сразу

Открой DevTools → Network и посмотри какой именно URL падает в 404.
С высокой вероятностью это один из этих двух.

Правильный фикс

Нужно не вести на файл шаблона компонента, а делать свой прокси-роут внутри компонента, который уже будет редиректить туда, куда реально умеет твоя сборка Bitrix.

1. Создай файл

/local/sitebuilder/components/disk/open.php

С кодом:

<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';
require_once __DIR__ . '/bootstrap.php';

use Bitrix\Disk\File;

try {
    $siteId = (int)($_GET['siteId'] ?? 0);
    $pageId = (int)($_GET['pageId'] ?? 0);
    $blockId = (int)($_GET['blockId'] ?? 0);
    $fileId = (int)($_GET['fileId'] ?? 0);
    $mode = (string)($_GET['mode'] ?? 'view');

    $currentUserId = DiskCurrentUser::requireId();

    $context = DiskContextFactory::fromArray([
        'siteId' => $siteId,
        'pageId' => $pageId,
        'blockId' => $blockId,
        'currentUserId' => $currentUserId,
    ]);

    DiskValidator::assertContext($context);

    $settings = DiskSettingsRepository::ensureExistsForBlock(
        $context->blockId,
        $context->siteId,
        $context->pageId,
        $context->currentUserId
    );

    $rootInfo = DiskRootResolver::resolveWithSource($context, $settings, false);
    $permissions = DiskPermissionService::resolve($context, $settings, $rootInfo['rootFolderId']);

    if (empty($permissions['canView'])) {
        throw new RuntimeException('ACCESS_DENIED');
    }

    $file = File::loadById($fileId);
    if (!$file instanceof File) {
        throw new RuntimeException('DISK_FILE_NOT_FOUND');
    }

    $fileName = (string)$file->getName();

    if ($mode === 'edit') {
        $url = '/disk/path/' . rawurlencode($fileName) . '?objectId=' . (int)$fileId . '&action=edit';
    } else {
        $url = '/disk/path/' . rawurlencode($fileName) . '?objectId=' . (int)$fileId;
    }

    LocalRedirect($url);
} catch (Throwable $e) {
    http_response_code(500);
    header('Content-Type: text/plain; charset=UTF-8');
    echo $e->getMessage();
    exit;
}


---

2. В DiskBitrixStorageAdapter.php замени методы построения office URL

Найди методы:

buildDiskOpenUrl

buildDiskEditUrl


И замени их на такие:

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


---

3. И не забудь обновить normalizeFile()

Если сейчас там вызовы без $context, должно быть так:

'previewUrl' => $isOffice
    ? $this->buildDiskOpenUrl($context, $file)
    : $this->buildPreviewUrl($context, $file),
'editUrl' => $isOffice
    ? $this->buildDiskEditUrl($context, $file)
    : '',


---

Почему это лучше

Потому что:

мы больше не дергаем несуществующий путь внутри /bitrix/components/...

у тебя появляется свой стабильный URL

вся проверка прав и контекста остается внутри компонента

дальше уже LocalRedirect() отправляет пользователя в штатный disk route


Если после этого снова будет 404

Тогда уже падает маршрут вида:

/disk/path/<filename>?objectId=...

И тогда надо будет посмотреть, какой именно disk route живой в твоей коробке.
В таком случае пришли полный URL из Network, который дал 404.