<?php

use Bitrix\Disk\Folder;

class SiteDiskInitializer
{
    /**
     * Укажи ID папки в "Общий диск", внутри которой нужно создавать папки сайтов.
     * Например, заранее руками создай в Общем диске папку "SiteBuilder" и подставь ее ID сюда.
     */
    protected const SHARED_ROOT_FOLDER_ID = 250;

    public static function ensureSiteRootFolder(int $siteId, int $currentUserId, string $siteName = ''): int
    {
        $existing = SiteRepository::getRootDiskFolderId($siteId);
        if ($existing !== null && $existing > 0) {
            return $existing;
        }

        $site = SiteRepository::getById($siteId);
        if (!$site) {
            throw new RuntimeException('SITE_NOT_FOUND');
        }

        $sharedRootFolder = self::getSharedRootFolder();
        if (!$sharedRootFolder instanceof Folder) {
            throw new RuntimeException('SHARED_DISK_ROOT_FOLDER_NOT_FOUND');
        }

        $folderBaseName = $siteName !== ''
            ? ('Сайт ' . $siteName)
            : ('Сайт ' . (string)$site['name']);

        $folderName = DiskNameSanitizer::sanitizeFolderName($folderBaseName, 'Сайт');

        $siteFolder = $sharedRootFolder->addSubFolder([
            'NAME' => $folderName,
            'CREATED_BY' => $currentUserId,
        ], [], true);

        if (!$siteFolder instanceof Folder) {
            $errors = [];

            if (method_exists($sharedRootFolder, 'getErrors')) {
                foreach ((array)$sharedRootFolder->getErrors() as $error) {
                    if (is_object($error) && method_exists($error, 'getMessage')) {
                        $errors[] = $error->getMessage();
                    } else {
                        $errors[] = (string)$error;
                    }
                }
            }

            throw new RuntimeException(
                'DISK_SITE_ROOT_CREATE_ERROR' . (!empty($errors) ? ': ' . implode(' | ', $errors) : '')
            );
        }

        SiteRepository::updateRootDiskFolderId($siteId, (int)$siteFolder->getId());

        return (int)$siteFolder->getId();
    }

    protected static function getSharedRootFolder(): Folder
    {
        $folderId = (int)self::SHARED_ROOT_FOLDER_ID;
        if ($folderId <= 0) {
            throw new RuntimeException('SHARED_ROOT_FOLDER_ID_NOT_CONFIGURED');
        }

        $folder = Folder::loadById($folderId);
        if (!$folder instanceof Folder) {
            throw new RuntimeException('SHARED_ROOT_FOLDER_NOT_FOUND');
        }

        return $folder;
    }
}


<?php

class DiskRootResolver
{
    public static function resolve(DiskContext $context, array $settings, bool $autoCreate = false): ?int
    {
        $result = self::resolveWithSource($context, $settings, $autoCreate);
        return $result['rootFolderId'];
    }

    public static function resolveWithSource(DiskContext $context, array $settings, bool $autoCreate = false): array
    {
        $rootMode = (string)($settings['rootMode'] ?? 'site');

        if ($rootMode === 'block') {
            if (!empty($settings['rootFolderId'])) {
                return [
                    'rootFolderId' => (int)$settings['rootFolderId'],
                    'source' => 'block',
                ];
            }

            if ($autoCreate) {
                $folderId = BlockDiskInitializer::ensureBlockRootFolder(
                    $context->siteId,
                    $context->pageId,
                    $context->blockId,
                    $context->currentUserId,
                    (string)($settings['title'] ?? '')
                );

                return [
                    'rootFolderId' => $folderId,
                    'source' => 'block',
                ];
            }
        }

        $siteRootFolderId = SiteRepository::getRootDiskFolderId($context->siteId);
        if ($siteRootFolderId !== null && $siteRootFolderId > 0) {
            return [
                'rootFolderId' => $siteRootFolderId,
                'source' => 'site',
            ];
        }

        if ($autoCreate && !empty($settings['useSiteRootFallback'])) {
            $site = SiteRepository::getById($context->siteId);
            if (!$site) {
                throw new RuntimeException('SITE_NOT_FOUND');
            }

            $folderId = SiteDiskInitializer::ensureSiteRootFolder(
                $context->siteId,
                $context->currentUserId,
                (string)$site['name']
            );

            return [
                'rootFolderId' => $folderId,
                'source' => 'site',
            ];
        }

        return [
            'rootFolderId' => null,
            'source' => 'none',
        ];
    }
}

<?php

class DiskSettingsRepository
{
    public static function getByBlockId(int $blockId): array
    {
        $block = DiskSitebuilderBridge::getBlockById($blockId);
        if (!$block) {
            throw new RuntimeException('BLOCK_NOT_FOUND');
        }

        return DiskSitebuilderBridge::normalizeDiskProps($block['props'] ?? []);
    }

    public static function createDefault(array $data): bool
    {
        $blockId = (int)($data['block_id'] ?? 0);
        if ($blockId <= 0) {
            throw new RuntimeException('INVALID_BLOCK_ID');
        }

        $default = DiskSitebuilderBridge::normalizeDiskProps([]);
        return DiskSitebuilderBridge::saveBlockProps($blockId, $default);
    }

    public static function save(int $blockId, array $settings): bool
    {
        $current = self::getByBlockId($blockId);
        $merged = array_merge($current, $settings);
        $normalized = DiskSitebuilderBridge::normalizeDiskProps($merged);

        return DiskSitebuilderBridge::saveBlockProps($blockId, $normalized);
    }

    public static function ensureExistsForBlock(int $blockId, int $siteId, int $pageId, ?int $createdBy = null): array
    {
        $block = DiskSitebuilderBridge::getBlockById($blockId);
        if (!$block) {
            throw new RuntimeException('BLOCK_NOT_FOUND');
        }

        $props = $block['props'] ?? [];
        $normalized = DiskSitebuilderBridge::normalizeDiskProps($props);

        if (($block['props'] ?? []) !== $normalized) {
            DiskSitebuilderBridge::saveBlockProps($blockId, $normalized);
        }

        return $normalized;
    }
}
