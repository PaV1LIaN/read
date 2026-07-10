<?php

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/PageAccessRepository.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/PageAccessService.php';

global $USER;

if (!function_exists('sb_page_access_json_success')) {
    function sb_page_access_json_success(array $data = []): void
    {
        header('Content-Type: application/json; charset=UTF-8');

        echo json_encode([
            'ok' => true,
            'data' => $data,
        ], JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES);

        exit;
    }
}

if (!function_exists('sb_page_access_json_error')) {
    function sb_page_access_json_error(string $message, int $status = 400, array $details = []): void
    {
        if (function_exists('sb_json_error')) {
            sb_json_error($message, $status, $details);
            exit;
        }

        http_response_code($status);
        header('Content-Type: application/json; charset=UTF-8');

        echo json_encode([
            'ok' => false,
            'error' => $message,
            'message' => $message,
            'details' => $details,
        ], JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES);

        exit;
    }
}

if (!function_exists('sb_page_access_bool')) {
    function sb_page_access_bool($value): bool
    {
        return $value === true
            || $value === 1
            || $value === '1'
            || $value === 'true'
            || $value === 'Y'
            || $value === 'on';
    }
}

if (!function_exists('sb_page_access_current_user_id')) {
    function sb_page_access_current_user_id(): int
    {
        global $USER;

        if (!is_object($USER) || !method_exists($USER, 'IsAuthorized') || !$USER->IsAuthorized()) {
            throw new RuntimeException('AUTH_REQUIRED');
        }

        return (int)$USER->GetID();
    }
}

/*
 * Пока управление правами страницы разрешаем:
 * 1. Админу Битрикса;
 * 2. Владельцу/админу сайта по старой глобальной модели;
 * 3. Пользователю, который может редактировать эту страницу.
 *
 * Позже можно заменить на отдельное право access.manage.
 */
if (!function_exists('sb_page_access_can_manage')) {
    function sb_page_access_can_manage(int $siteId, int $pageId, int $userId): bool
    {
        global $USER;

        if (is_object($USER) && method_exists($USER, 'IsAdmin') && $USER->IsAdmin()) {
            return true;
        }

        if (PageAccessService::hasGlobalSiteAccess($siteId, $userId, 'admin')) {
            return true;
        }

        return PageAccessService::canEditPage($siteId, $pageId, $userId);
    }
}

try {
    $action = (string)($_POST['action'] ?? '');

    $currentUserId = sb_page_access_current_user_id();

    if ($action === 'pageAccess.list') {
        $siteId = (int)($_POST['siteId'] ?? 0);
        $pageId = (int)($_POST['pageId'] ?? 0);

        if ($siteId <= 0) {
            throw new RuntimeException('INVALID_SITE_ID');
        }

        if ($pageId <= 0) {
            throw new RuntimeException('INVALID_PAGE_ID');
        }

        if (!sb_page_access_can_manage($siteId, $pageId, $currentUserId)) {
            throw new RuntimeException('PAGE_ACCESS_DENIED');
        }

        $items = PageAccessRepository::listByPage($siteId, $pageId);

        sb_page_access_json_success([
            'items' => $items,
        ]);
    }

    if ($action === 'pageAccess.save') {
        if (!check_bitrix_sessid()) {
            throw new RuntimeException('BAD_SESSID');
        }

        $siteId = (int)($_POST['siteId'] ?? 0);
        $pageId = (int)($_POST['pageId'] ?? 0);
        $accessCode = (string)($_POST['accessCode'] ?? '');

        $canView = sb_page_access_bool($_POST['canView'] ?? false);
        $canEdit = sb_page_access_bool($_POST['canEdit'] ?? false);
        $includeChildren = sb_page_access_bool($_POST['includeChildren'] ?? false);

        if ($siteId <= 0) {
            throw new RuntimeException('INVALID_SITE_ID');
        }

        if ($pageId <= 0) {
            throw new RuntimeException('INVALID_PAGE_ID');
        }

        if ($accessCode === '') {
            throw new RuntimeException('EMPTY_ACCESS_CODE');
        }

        if (!$canView && !$canEdit) {
            throw new RuntimeException('EMPTY_PAGE_PERMISSION');
        }

        /*
         * Редактирование автоматически включает чтение.
         */
        if ($canEdit) {
            $canView = true;
        }

        if (!sb_page_access_can_manage($siteId, $pageId, $currentUserId)) {
            throw new RuntimeException('PAGE_ACCESS_DENIED');
        }

        $item = PageAccessRepository::save(
            $siteId,
            $pageId,
            $accessCode,
            $canView,
            $canEdit,
            $includeChildren,
            $currentUserId
        );

        sb_page_access_json_success([
            'item' => $item,
        ]);
    }

    if ($action === 'pageAccess.delete') {
        if (!check_bitrix_sessid()) {
            throw new RuntimeException('BAD_SESSID');
        }

        $id = (int)($_POST['id'] ?? 0);
        $siteId = (int)($_POST['siteId'] ?? 0);
        $pageId = (int)($_POST['pageId'] ?? 0);

        if ($id <= 0) {
            throw new RuntimeException('INVALID_PAGE_ACCESS_ID');
        }

        if ($siteId <= 0) {
            throw new RuntimeException('INVALID_SITE_ID');
        }

        if ($pageId <= 0) {
            throw new RuntimeException('INVALID_PAGE_ID');
        }

        if (!sb_page_access_can_manage($siteId, $pageId, $currentUserId)) {
            throw new RuntimeException('PAGE_ACCESS_DENIED');
        }

        PageAccessRepository::delete($id, $siteId, $pageId);

        sb_page_access_json_success([
            'deleted' => true,
            'id' => $id,
        ]);
    }

    throw new RuntimeException('UNKNOWN_PAGE_ACCESS_ACTION');
} catch (Throwable $e) {
    sb_page_access_json_error($e->getMessage(), 400, [
        'action' => (string)($_POST['action'] ?? ''),
        'file' => $e->getFile(),
        'line' => $e->getLine(),
    ]);
}
