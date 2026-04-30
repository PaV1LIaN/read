Да, тогда фиксируем точку отката и идём маленькими шагами.

Точка отката перед изменениями

На сервере перед правками сделай копию проекта:

cd /srv/bx/docroot/local

mkdir -p /srv/bx/rollback/sitebuilder_before_appearance

cp -a /srv/bx/docroot/local/sitebuilder /srv/bx/rollback/sitebuilder_before_appearance/sitebuilder

Если не понравится, откат будет такой:

rm -rf /srv/bx/docroot/local/sitebuilder
cp -a /srv/bx/rollback/sitebuilder_before_appearance/sitebuilder /srv/bx/docroot/local/sitebuilder

Теперь делаем первый этап: backend для логотипа и фона через CFile, без Диска.


---

1. Создай файл

/local/sitebuilder/lib/SiteAppearanceService.php

<?php

class SiteAppearanceService
{
    protected const UPLOAD_DIR = 'sitebuilder/appearance';

    protected const MAX_FILE_SIZE = 10485760; // 10 MB

    protected const ALLOWED_EXTENSIONS = [
        'jpg',
        'jpeg',
        'png',
        'webp',
        'gif',
    ];

    protected const ALLOWED_MIME_TYPES = [
        'image/jpeg',
        'image/png',
        'image/webp',
        'image/gif',
    ];

    public static function get(int $siteId): array
    {
        $site = self::getSiteOrFail($siteId);
        $settings = self::normalizeAppearanceSettings($site['settings'] ?? []);

        return self::withUrls($settings);
    }

    public static function update(int $siteId, array $data, int $currentUserId): array
    {
        $site = self::getSiteOrFail($siteId);
        $settings = self::normalizeAppearanceSettings($site['settings'] ?? []);

        if (array_key_exists('backgroundColor', $data)) {
            $settings['backgroundColor'] = self::normalizeColor((string)$data['backgroundColor'], '#f8fafc');
        }

        if (array_key_exists('backgroundMode', $data)) {
            $settings['backgroundMode'] = self::normalizeBackgroundMode((string)$data['backgroundMode']);
        }

        if (array_key_exists('backgroundPosition', $data)) {
            $settings['backgroundPosition'] = self::normalizeBackgroundPosition((string)$data['backgroundPosition']);
        }

        if (array_key_exists('backgroundRepeat', $data)) {
            $settings['backgroundRepeat'] = self::normalizeBackgroundRepeat((string)$data['backgroundRepeat']);
        }

        if (array_key_exists('headerLogoMode', $data)) {
            $settings['headerLogoMode'] = self::normalizeHeaderLogoMode((string)$data['headerLogoMode']);
        }

        self::saveSiteSettings($siteId, $settings, $currentUserId);

        return self::withUrls($settings);
    }

    public static function upload(int $siteId, string $type, array $file, int $currentUserId): array
    {
        $type = self::normalizeAssetType($type);

        if (!class_exists('CFile')) {
            throw new RuntimeException('CFile_NOT_FOUND');
        }

        self::validateUploadFile($file);

        $site = self::getSiteOrFail($siteId);
        $settings = self::normalizeAppearanceSettings($site['settings'] ?? []);

        $oldFileIdKey = $type === 'logo' ? 'logoFileId' : 'backgroundFileId';
        $oldFileId = (int)($settings[$oldFileIdKey] ?? 0);

        $file['MODULE_ID'] = 'main';

        $newFileId = (int)CFile::SaveFile($file, self::UPLOAD_DIR);

        if ($newFileId <= 0) {
            throw new RuntimeException('FILE_SAVE_ERROR');
        }

        if ($oldFileId > 0) {
            CFile::Delete($oldFileId);
        }

        $settings[$oldFileIdKey] = $newFileId;

        self::saveSiteSettings($siteId, $settings, $currentUserId);

        return self::withUrls($settings);
    }

    public static function remove(int $siteId, string $type, int $currentUserId): array
    {
        $type = self::normalizeAssetType($type);

        if (!class_exists('CFile')) {
            throw new RuntimeException('CFile_NOT_FOUND');
        }

        $site = self::getSiteOrFail($siteId);
        $settings = self::normalizeAppearanceSettings($site['settings'] ?? []);

        $fileIdKey = $type === 'logo' ? 'logoFileId' : 'backgroundFileId';
        $fileId = (int)($settings[$fileIdKey] ?? 0);

        if ($fileId > 0) {
            CFile::Delete($fileId);
        }

        $settings[$fileIdKey] = 0;

        self::saveSiteSettings($siteId, $settings, $currentUserId);

        return self::withUrls($settings);
    }

    protected static function getSiteOrFail(int $siteId): array
    {
        if ($siteId <= 0) {
            throw new RuntimeException('EMPTY_SITE_ID');
        }

        if (!function_exists('sb_read_sites')) {
            throw new RuntimeException('STORAGE_NOT_LOADED');
        }

        $sites = sb_read_sites();

        foreach ($sites as $site) {
            if ((int)($site['id'] ?? 0) === $siteId) {
                return $site;
            }
        }

        throw new RuntimeException('SITE_NOT_FOUND');
    }

    protected static function saveSiteSettings(int $siteId, array $settings, int $currentUserId): void
    {
        if (!function_exists('sb_read_sites') || !function_exists('sb_write_sites')) {
            throw new RuntimeException('STORAGE_NOT_LOADED');
        }

        $sites = sb_read_sites();
        $found = false;

        foreach ($sites as &$site) {
            if ((int)($site['id'] ?? 0) !== $siteId) {
                continue;
            }

            $currentSettings = is_array($site['settings'] ?? null) ? $site['settings'] : [];
            $site['settings'] = array_merge($currentSettings, $settings);
            $site['updatedBy'] = $currentUserId;
            $site['updatedAt'] = date('c');

            $found = true;
            break;
        }
        unset($site);

        if (!$found) {
            throw new RuntimeException('SITE_NOT_FOUND');
        }

        sb_write_sites($sites);
    }

    protected static function normalizeAppearanceSettings(array $settings): array
    {
        return [
            'accent' => (string)($settings['accent'] ?? '#2563eb'),

            'logoFileId' => (int)($settings['logoFileId'] ?? 0),
            'backgroundFileId' => (int)($settings['backgroundFileId'] ?? 0),

            'backgroundColor' => self::normalizeColor(
                (string)($settings['backgroundColor'] ?? '#f8fafc'),
                '#f8fafc'
            ),

            'backgroundMode' => self::normalizeBackgroundMode(
                (string)($settings['backgroundMode'] ?? 'cover')
            ),

            'backgroundPosition' => self::normalizeBackgroundPosition(
                (string)($settings['backgroundPosition'] ?? 'center center')
            ),

            'backgroundRepeat' => self::normalizeBackgroundRepeat(
                (string)($settings['backgroundRepeat'] ?? 'no-repeat')
            ),

            'headerLogoMode' => self::normalizeHeaderLogoMode(
                (string)($settings['headerLogoMode'] ?? 'image')
            ),
        ];
    }

    protected static function withUrls(array $settings): array
    {
        $settings['logoUrl'] = '';
        $settings['backgroundUrl'] = '';

        if (class_exists('CFile')) {
            if (!empty($settings['logoFileId'])) {
                $settings['logoUrl'] = (string)CFile::GetPath((int)$settings['logoFileId']);
            }

            if (!empty($settings['backgroundFileId'])) {
                $settings['backgroundUrl'] = (string)CFile::GetPath((int)$settings['backgroundFileId']);
            }
        }

        return $settings;
    }

    protected static function validateUploadFile(array $file): void
    {
        $error = (int)($file['error'] ?? UPLOAD_ERR_NO_FILE);

        if ($error !== UPLOAD_ERR_OK) {
            throw new RuntimeException('UPLOAD_ERROR_' . $error);
        }

        $size = (int)($file['size'] ?? 0);

        if ($size <= 0) {
            throw new RuntimeException('EMPTY_FILE');
        }

        if ($size > self::MAX_FILE_SIZE) {
            throw new RuntimeException('FILE_TOO_LARGE');
        }

        $name = (string)($file['name'] ?? '');
        $ext = strtolower(pathinfo($name, PATHINFO_EXTENSION));

        if (!in_array($ext, self::ALLOWED_EXTENSIONS, true)) {
            throw new RuntimeException('BAD_FILE_EXTENSION');
        }

        $mime = '';

        if (!empty($file['type'])) {
            $mime = strtolower((string)$file['type']);
        }

        if ($mime !== '' && !in_array($mime, self::ALLOWED_MIME_TYPES, true)) {
            throw new RuntimeException('BAD_FILE_MIME_TYPE');
        }
    }

    protected static function normalizeAssetType(string $type): string
    {
        $type = trim($type);

        if (!in_array($type, ['logo', 'background'], true)) {
            throw new RuntimeException('BAD_ASSET_TYPE');
        }

        return $type;
    }

    protected static function normalizeColor(string $color, string $fallback): string
    {
        $color = trim($color);

        if (preg_match('/^#[0-9a-fA-F]{6}$/', $color)) {
            return strtolower($color);
        }

        if (preg_match('/^#[0-9a-fA-F]{3}$/', $color)) {
            return strtolower($color);
        }

        return $fallback;
    }

    protected static function normalizeBackgroundMode(string $mode): string
    {
        $mode = trim($mode);

        $allowed = [
            'cover',
            'contain',
            'auto',
            'stretch',
        ];

        return in_array($mode, $allowed, true) ? $mode : 'cover';
    }

    protected static function normalizeBackgroundPosition(string $position): string
    {
        $position = trim($position);

        $allowed = [
            'center center',
            'top center',
            'bottom center',
            'left center',
            'right center',
        ];

        return in_array($position, $allowed, true) ? $position : 'center center';
    }

    protected static function normalizeBackgroundRepeat(string $repeat): string
    {
        $repeat = trim($repeat);

        $allowed = [
            'no-repeat',
            'repeat',
            'repeat-x',
            'repeat-y',
        ];

        return in_array($repeat, $allowed, true) ? $repeat : 'no-repeat';
    }

    protected static function normalizeHeaderLogoMode(string $mode): string
    {
        $mode = trim($mode);

        $allowed = [
            'image',
            'text',
            'both',
        ];

        return in_array($mode, $allowed, true) ? $mode : 'image';
    }
}


---

2. В /local/sitebuilder/api/index.php

В блок site-действий добавь 4 действия:

$action === 'site.appearanceGet' ||
$action === 'site.appearanceUpdate' ||
$action === 'site.appearanceUpload' ||
$action === 'site.appearanceRemove' ||

Должно быть примерно так:

if (
    $action === 'site.list' ||
    $action === 'site.get' ||
    $action === 'site.create' ||
    $action === 'site.update' ||
    $action === 'site.delete' ||
    $action === 'site.setHome' ||
    $action === 'site.syncAccess' ||
    $action === 'site.ensureGroup' ||
    $action === 'site.accessList' ||
    $action === 'site.accessSet' ||
    $action === 'site.accessRemove' ||
    $action === 'site.appearanceGet' ||
    $action === 'site.appearanceUpdate' ||
    $action === 'site.appearanceUpload' ||
    $action === 'site.appearanceRemove'
) {
    require __DIR__ . '/handlers/site.php';
    exit;
}


---

3. В /local/sitebuilder/api/handlers/site.php

В самый верх, после <?php, добавь:

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/SiteAppearanceService.php';

Потом ближе к низу файла, до финального NOT_MOVED_YET, добавь:

if ($action === 'site.appearanceGet') {
    $siteId = (int)($_POST['siteId'] ?? 0);

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    sb_require_content_manager($siteId);

    try {
        $appearance = SiteAppearanceService::get($siteId);

        sb_json_ok([
            'appearance' => $appearance,
            'handler' => 'site',
            'action' => $action,
            'file' => __FILE__,
        ]);
    } catch (Throwable $e) {
        sb_json_error($e->getMessage(), 500, [
            'handler' => 'site',
            'action' => $action,
            'file' => __FILE__,
        ]);
    }
}

if ($action === 'site.appearanceUpdate') {
    global $USER;

    $siteId = (int)($_POST['siteId'] ?? 0);

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    sb_require_content_manager($siteId);

    try {
        $data = $_POST;

        unset($data['action'], $data['sessid'], $data['siteId']);

        $appearance = SiteAppearanceService::update(
            $siteId,
            $data,
            (int)$USER->GetID()
        );

        sb_json_ok([
            'appearance' => $appearance,
            'handler' => 'site',
            'action' => $action,
            'file' => __FILE__,
        ]);
    } catch (Throwable $e) {
        sb_json_error($e->getMessage(), 500, [
            'handler' => 'site',
            'action' => $action,
            'file' => __FILE__,
        ]);
    }
}

if ($action === 'site.appearanceUpload') {
    global $USER;

    $siteId = (int)($_POST['siteId'] ?? 0);
    $type = trim((string)($_POST['type'] ?? ''));

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    if ($type === '') {
        sb_json_error('TYPE_REQUIRED', 422);
    }

    sb_require_content_manager($siteId);

    $file = $_FILES['file'] ?? null;

    if (!is_array($file)) {
        sb_json_error('FILE_REQUIRED', 422);
    }

    try {
        $appearance = SiteAppearanceService::upload(
            $siteId,
            $type,
            $file,
            (int)$USER->GetID()
        );

        sb_json_ok([
            'appearance' => $appearance,
            'handler' => 'site',
            'action' => $action,
            'file' => __FILE__,
        ]);
    } catch (Throwable $e) {
        sb_json_error($e->getMessage(), 500, [
            'handler' => 'site',
            'action' => $action,
            'file' => __FILE__,
        ]);
    }
}

if ($action === 'site.appearanceRemove') {
    global $USER;

    $siteId = (int)($_POST['siteId'] ?? 0);
    $type = trim((string)($_POST['type'] ?? ''));

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    if ($type === '') {
        sb_json_error('TYPE_REQUIRED', 422);
    }

    sb_require_content_manager($siteId);

    try {
        $appearance = SiteAppearanceService::remove(
            $siteId,
            $type,
            (int)$USER->GetID()
        );

        sb_json_ok([
            'appearance' => $appearance,
            'handler' => 'site',
            'action' => $action,
            'file' => __FILE__,
        ]);
    } catch (Throwable $e) {
        sb_json_error($e->getMessage(), 500, [
            'handler' => 'site',
            'action' => $action,
            'file' => __FILE__,
        ]);
    }
}


---

4. Быстрая проверка из консоли браузера

Открой editor.php?siteId=13 и выполни:

fetch('/local/sitebuilder/api.php', {
  method: 'POST',
  body: new URLSearchParams({
    action: 'site.appearanceGet',
    siteId: '13',
    sessid: BX.bitrix_sessid()
  }),
  credentials: 'same-origin'
})
.then(r => r.json())
.then(console.log)
.catch(console.error);

Должен прийти примерно такой ответ:

{
  "ok": true,
  "appearance": {
    "accent": "#2563eb",
    "logoFileId": 0,
    "backgroundFileId": 0,
    "backgroundColor": "#f8fafc",
    "backgroundMode": "cover",
    "backgroundPosition": "center center",
    "backgroundRepeat": "no-repeat",
    "headerLogoMode": "image",
    "logoUrl": "",
    "backgroundUrl": ""
  }
}

Если это сработает, следующим шагом добавим интерфейс в settings.php: загрузка логотипа, загрузка фона, цвет фона и режим отображения.