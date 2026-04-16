Ниже следующие файлы после bootstrap.php.


---

/local/sitebuilder/components/disk/class.php

<?php

require_once __DIR__ . '/bootstrap.php';

class SitebuilderDiskComponent
{
    protected array $params = [];
    protected array $result = [];

    public function __construct(array $params = [])
    {
        $this->params = $params;
    }

    public function execute(): void
    {
        try {
            $context = DiskContextFactory::fromArray([
                'siteId' => (int)($this->params['SITE_ID'] ?? 0),
                'pageId' => (int)($this->params['PAGE_ID'] ?? 0),
                'blockId' => (int)($this->params['BLOCK_ID'] ?? 0),
                'currentUserId' => (int)($this->params['CURRENT_USER_ID'] ?? 0),
            ]);

            DiskValidator::assertContext($context);

            $settings = DiskSettingsRepository::ensureExistsForBlock(
                $context->blockId,
                $context->siteId,
                $context->pageId,
                $context->currentUserId
            );

            $root = DiskRootResolver::resolveWithSource($context, $settings);
            $permissions = DiskPermissionService::resolve($context, $settings, $root['rootFolderId']);

            $this->result = [
                'SITE_ID' => $context->siteId,
                'PAGE_ID' => $context->pageId,
                'BLOCK_ID' => $context->blockId,
                'CURRENT_USER_ID' => $context->currentUserId,
                'SETTINGS' => $settings,
                'ROOT_FOLDER_ID' => $root['rootFolderId'],
                'ROOT_SOURCE' => $root['source'],
                'PERMISSIONS' => $permissions,
                'TITLE' => $settings['title'] ?? 'Файлы',
                'INITIAL_STATE' => [
                    'siteId' => $context->siteId,
                    'pageId' => $context->pageId,
                    'blockId' => $context->blockId,
                    'rootFolderId' => $root['rootFolderId'],
                    'rootSource' => $root['source'],
                    'currentFolderId' => $root['rootFolderId'],
                    'settings' => $settings,
                    'permissions' => $permissions,
                ],
                'ERROR' => null,
            ];
        } catch (Throwable $e) {
            $this->result = [
                'SITE_ID' => (int)($this->params['SITE_ID'] ?? 0),
                'PAGE_ID' => (int)($this->params['PAGE_ID'] ?? 0),
                'BLOCK_ID' => (int)($this->params['BLOCK_ID'] ?? 0),
                'CURRENT_USER_ID' => (int)($this->params['CURRENT_USER_ID'] ?? 0),
                'SETTINGS' => [],
                'ROOT_FOLDER_ID' => null,
                'ROOT_SOURCE' => 'none',
                'PERMISSIONS' => [
                    'canView' => false,
                    'canUpload' => false,
                    'canCreateFolder' => false,
                    'canRename' => false,
                    'canDelete' => false,
                    'canDownload' => false,
                    'canManageAccess' => false,
                    'canEditSettings' => false,
                ],
                'TITLE' => 'Файлы',
                'INITIAL_STATE' => [
                    'siteId' => (int)($this->params['SITE_ID'] ?? 0),
                    'pageId' => (int)($this->params['PAGE_ID'] ?? 0),
                    'blockId' => (int)($this->params['BLOCK_ID'] ?? 0),
                    'rootFolderId' => null,
                    'rootSource' => 'none',
                    'currentFolderId' => null,
                    'settings' => [],
                    'permissions' => [
                        'canView' => false,
                        'canUpload' => false,
                        'canCreateFolder' => false,
                        'canRename' => false,
                        'canDelete' => false,
                        'canDownload' => false,
                        'canManageAccess' => false,
                        'canEditSettings' => false,
                    ],
                ],
                'ERROR' => $e->getMessage(),
            ];
        }

        $arResult = $this->result;
        include __DIR__ . '/template.php';
    }
}


---

/local/sitebuilder/components/disk/api.php

<?php

require_once __DIR__ . '/bootstrap.php';

try {
    $action = $_GET['action'] ?? '';

    switch ($action) {
        case 'resolveRoot':
            require __DIR__ . '/actions/resolve_root.php';
            break;

        case 'getSettings':
            require __DIR__ . '/actions/get_settings.php';
            break;

        case 'saveSettings':
            require __DIR__ . '/actions/save_settings.php';
            break;

        case 'getPermissions':
            require __DIR__ . '/actions/get_permissions.php';
            break;

        case 'getRootOptions':
            require __DIR__ . '/actions/get_root_options.php';
            break;

        case 'list':
            require __DIR__ . '/actions/list.php';
            break;

        case 'upload':
            require __DIR__ . '/actions/upload.php';
            break;

        case 'createFolder':
            require __DIR__ . '/actions/create_folder.php';
            break;

        case 'rename':
            require __DIR__ . '/actions/rename.php';
            break;

        case 'delete':
            require __DIR__ . '/actions/delete.php';
            break;

        case 'move':
            require __DIR__ . '/actions/move.php';
            break;

        case 'copy':
            require __DIR__ . '/actions/copy.php';
            break;

        case 'search':
            require __DIR__ . '/actions/search.php';
            break;

        case 'download':
            require __DIR__ . '/actions/download.php';
            break;

        case 'initSiteRoot':
            require __DIR__ . '/actions/init_site_root.php';
            break;

        case 'initBlockRoot':
            require __DIR__ . '/actions/init_block_root.php';
            break;

        default:
            DiskResponse::error('UNKNOWN_ACTION', 'Неизвестное действие');
    }
} catch (Throwable $e) {
    DiskResponse::error('SERVER_ERROR', $e->getMessage());
}


---

/local/sitebuilder/components/disk/lib/helpers.php

<?php

function disk_h(?string $value): string
{
    return htmlspecialchars((string)$value, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
}

function disk_read_json_body(): array
{
    static $cached = null;

    if ($cached !== null) {
        return $cached;
    }

    $raw = file_get_contents('php://input');
    if ($raw === false || $raw === '') {
        $cached = [];
        return $cached;
    }

    $data = json_decode($raw, true);
    $cached = is_array($data) ? $data : [];

    return $cached;
}

function disk_normalize_bool($value): bool
{
    if (is_bool($value)) {
        return $value;
    }

    if (is_int($value)) {
        return $value === 1;
    }

    if (is_string($value)) {
        $value = strtolower(trim($value));
        return in_array($value, ['1', 'y', 'yes', 'true', 'on'], true);
    }

    return false;
}


---

/local/sitebuilder/components/disk/lib/DiskContext.php

<?php

class DiskContext
{
    public int $siteId;
    public int $pageId;
    public int $blockId;
    public int $currentUserId;

    public function __construct(int $siteId, int $pageId, int $blockId, int $currentUserId)
    {
        $this->siteId = $siteId;
        $this->pageId = $pageId;
        $this->blockId = $blockId;
        $this->currentUserId = $currentUserId;
    }

    public function toArray(): array
    {
        return [
            'siteId' => $this->siteId,
            'pageId' => $this->pageId,
            'blockId' => $this->blockId,
            'currentUserId' => $this->currentUserId,
        ];
    }
}

class DiskContextFactory
{
    public static function fromArray(array $data): DiskContext
    {
        return new DiskContext(
            (int)($data['siteId'] ?? 0),
            (int)($data['pageId'] ?? 0),
            (int)($data['blockId'] ?? 0),
            (int)($data['currentUserId'] ?? 0)
        );
    }
}


---

/local/sitebuilder/components/disk/lib/DiskResponse.php

<?php

class DiskResponse
{
    public static function success(array $data = [], array $meta = []): void
    {
        self::send([
            'ok' => true,
            'data' => $data,
            'meta' => $meta,
            'error' => null,
            'message' => '',
        ]);
    }

    public static function error(string $errorCode, string $message = '', array $details = []): void
    {
        self::send([
            'ok' => false,
            'data' => [],
            'meta' => [],
            'error' => $errorCode,
            'message' => $message,
            'details' => $details,
        ]);
    }

    protected static function send(array $payload): void
    {
        header('Content-Type: application/json; charset=UTF-8');
        echo json_encode($payload, JSON_UNESCAPED_UNICODE);
        exit;
    }
}


---

/local/sitebuilder/components/disk/lib/DiskCurrentUser.php

<?php

class DiskCurrentUser
{
    public static function getId(): int
    {
        global $USER;

        if ($USER instanceof CUser) {
            return (int)$USER->GetID();
        }

        return 0;
    }

    public static function requireId(): int
    {
        $userId = self::getId();
        if ($userId <= 0) {
            throw new RuntimeException('NOT_AUTHORIZED');
        }

        return $userId;
    }

    public static function isAdmin(): bool
    {
        global $USER;

        if (!($USER instanceof CUser)) {
            return false;
        }

        return $USER->IsAdmin();
    }

    public static function getGroupIds(): array
    {
        global $USER;

        if (!($USER instanceof CUser)) {
            return [];
        }

        $groups = $USER->GetUserGroupArray();
        if (!is_array($groups)) {
            return [];
        }

        return array_values(array_map('intval', $groups));
    }
}


---

/local/sitebuilder/components/disk/lib/DiskCsrf.php

<?php

class DiskCsrf
{
    public static function validateFromRequest(): void
    {
        $sessid = '';

        if (isset($_POST['sessid'])) {
            $sessid = (string)$_POST['sessid'];
        } elseif (isset($_REQUEST['sessid'])) {
            $sessid = (string)$_REQUEST['sessid'];
        } else {
            $json = disk_read_json_body();
            if (isset($json['sessid'])) {
                $sessid = (string)$json['sessid'];
            }
        }

        if ($sessid === '') {
            throw new RuntimeException('EMPTY_SESSID');
        }

        if (!check_bitrix_sessid($sessid)) {
            throw new RuntimeException('BAD_SESSID');
        }
    }
}

Следом пришлю DiskDb.php, BlockRepository.php, SiteRepository.php, SiteAccessRepository.php, DiskSettingsRepository.php.