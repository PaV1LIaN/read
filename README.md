Полный обновлённый файл:

/local/sitebuilder/lib/PageAccessRepository.php

<?php

require_once __DIR__ . '/db.php';

class PageAccessRepository
{
    public static function userAccessCode(int $userId): string
    {
        if ($userId <= 0) {
            throw new RuntimeException('INVALID_USER_ID');
        }

        return 'U' . $userId;
    }

    public static function normalizeAccessCode(string $accessCode): string
    {
        $accessCode = mb_strtoupper(trim($accessCode));

        if ($accessCode === '') {
            throw new RuntimeException('EMPTY_ACCESS_CODE');
        }

        /*
         * U123 — пользователь Битрикс24.
         * G123 — группа. Оставлено для будущего расширения.
         */
        if (!preg_match('/^(U|G)[1-9]\d*$/', $accessCode)) {
            throw new RuntimeException('INVALID_ACCESS_CODE');
        }

        return $accessCode;
    }

    /**
     * Проверяет, что страница действительно принадлежит указанному сайту.
     */
    public static function pageBelongsToSite(int $siteId, int $pageId): bool
    {
        if ($siteId <= 0 || $pageId <= 0) {
            return false;
        }

        $pdo = sb_db();

        $stmt = $pdo->prepare("
            SELECT 1
            FROM sitebuilder.page
            WHERE site_id = :site_id
              AND id = :page_id
            LIMIT 1
        ");

        $stmt->execute([
            ':site_id' => $siteId,
            ':page_id' => $pageId,
        ]);

        return (bool)$stmt->fetchColumn();
    }

    /**
     * Прерывает выполнение, если страница не принадлежит сайту.
     */
    public static function requirePageInSite(int $siteId, int $pageId): void
    {
        if ($siteId <= 0) {
            throw new RuntimeException('INVALID_SITE_ID');
        }

        if ($pageId <= 0) {
            throw new RuntimeException('INVALID_PAGE_ID');
        }

        if (!self::pageBelongsToSite($siteId, $pageId)) {
            throw new RuntimeException('PAGE_NOT_IN_SITE');
        }
    }

    public static function listByPage(int $siteId, int $pageId): array
    {
        if ($siteId <= 0 || $pageId <= 0) {
            return [];
        }

        self::requirePageInSite($siteId, $pageId);

        $pdo = sb_db();

        $stmt = $pdo->prepare("
            SELECT
                id,
                site_id,
                page_id,
                access_code,
                can_view,
                can_edit,
                include_children,
                created_by,
                created_at,
                updated_at
            FROM sitebuilder.page_access
            WHERE site_id = :site_id
              AND page_id = :page_id
            ORDER BY id DESC
        ");

        $stmt->execute([
            ':site_id' => $siteId,
            ':page_id' => $pageId,
        ]);

        $rows = $stmt->fetchAll(PDO::FETCH_ASSOC);

        return array_map(
            [self::class, 'mapRow'],
            $rows ?: []
        );
    }

    public static function save(
        int $siteId,
        int $pageId,
        string $accessCode,
        bool $canView,
        bool $canEdit,
        bool $includeChildren,
        int $createdBy = 0
    ): array {
        if ($siteId <= 0) {
            throw new RuntimeException('INVALID_SITE_ID');
        }

        if ($pageId <= 0) {
            throw new RuntimeException('INVALID_PAGE_ID');
        }

        self::requirePageInSite($siteId, $pageId);

        $accessCode = self::normalizeAccessCode($accessCode);

        /*
         * Редактирование всегда включает просмотр.
         */
        if ($canEdit) {
            $canView = true;
        }

        if (!$canView && !$canEdit) {
            throw new RuntimeException('EMPTY_PAGE_PERMISSION');
        }

        $pdo = sb_db();

        $stmt = $pdo->prepare("
            INSERT INTO sitebuilder.page_access (
                site_id,
                page_id,
                access_code,
                can_view,
                can_edit,
                include_children,
                created_by,
                created_at,
                updated_at
            )
            VALUES (
                :site_id,
                :page_id,
                :access_code,
                :can_view,
                :can_edit,
                :include_children,
                :created_by,
                NOW(),
                NOW()
            )
            ON CONFLICT (site_id, page_id, access_code)
            DO UPDATE SET
                can_view = EXCLUDED.can_view,
                can_edit = EXCLUDED.can_edit,
                include_children = EXCLUDED.include_children,
                updated_at = NOW()
            RETURNING
                id,
                site_id,
                page_id,
                access_code,
                can_view,
                can_edit,
                include_children,
                created_by,
                created_at,
                updated_at
        ");

        $stmt->execute([
            ':site_id' => $siteId,
            ':page_id' => $pageId,
            ':access_code' => $accessCode,
            ':can_view' => $canView,
            ':can_edit' => $canEdit,
            ':include_children' => $includeChildren,
            ':created_by' => $createdBy > 0 ? $createdBy : null,
        ]);

        $row = $stmt->fetch(PDO::FETCH_ASSOC);

        if (!$row) {
            throw new RuntimeException('PAGE_ACCESS_SAVE_ERROR');
        }

        return self::mapRow($row);
    }

    public static function delete(
        int $id,
        int $siteId = 0,
        int $pageId = 0
    ): bool {
        if ($id <= 0) {
            throw new RuntimeException('INVALID_PAGE_ACCESS_ID');
        }

        $pdo = sb_db();

        $where = [
            'id = :id',
        ];

        $params = [
            ':id' => $id,
        ];

        if ($siteId > 0) {
            $where[] = 'site_id = :site_id';
            $params[':site_id'] = $siteId;
        }

        if ($pageId > 0) {
            $where[] = 'page_id = :page_id';
            $params[':page_id'] = $pageId;
        }

        $stmt = $pdo->prepare("
            DELETE FROM sitebuilder.page_access
            WHERE " . implode(' AND ', $where)
        );

        $stmt->execute($params);

        return $stmt->rowCount() > 0;
    }

    public static function hasAnyPageAccess(
        int $siteId,
        string $accessCode
    ): bool {
        if ($siteId <= 0) {
            return false;
        }

        $accessCode = self::normalizeAccessCode($accessCode);

        $pdo = sb_db();

        $stmt = $pdo->prepare("
            SELECT 1
            FROM sitebuilder.page_access
            WHERE site_id = :site_id
              AND access_code = :access_code
              AND (
                    can_view = TRUE
                 OR can_edit = TRUE
              )
            LIMIT 1
        ");

        $stmt->execute([
            ':site_id' => $siteId,
            ':access_code' => $accessCode,
        ]);

        return (bool)$stmt->fetchColumn();
    }

    public static function hasPagePermission(
        int $siteId,
        int $pageId,
        string $accessCode,
        string $permission
    ): bool {
        if ($siteId <= 0 || $pageId <= 0) {
            return false;
        }

        $accessCode = self::normalizeAccessCode($accessCode);

        if (!in_array($permission, ['view', 'edit'], true)) {
            return false;
        }

        /*
         * Возвращается цепочка:
         * текущая страница → родитель → родитель родителя.
         */
        $pageAndParents = self::getPageAndParentIds(
            $siteId,
            $pageId
        );

        if (empty($pageAndParents)) {
            return false;
        }

        $placeholders = [];

        foreach ($pageAndParents as $index => $id) {
            $placeholders[] = ':page_id_' . $index;
        }

        $params = [
            ':site_id' => $siteId,
            ':access_code' => $accessCode,
        ];

        foreach ($pageAndParents as $index => $id) {
            $params[':page_id_' . $index] = (int)$id;
        }

        $pdo = sb_db();

        $stmt = $pdo->prepare("
            SELECT
                page_id,
                can_view,
                can_edit,
                include_children
            FROM sitebuilder.page_access
            WHERE site_id = :site_id
              AND access_code = :access_code
              AND page_id IN (" . implode(',', $placeholders) . ")
        ");

        $stmt->execute($params);

        $rulesByPageId = [];

        while ($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
            $rulesByPageId[(int)$row['page_id']] = [
                'canView' => self::boolValue($row['can_view']),
                'canEdit' => self::boolValue($row['can_edit']),
                'includeChildren' => self::boolValue(
                    $row['include_children']
                ),
            ];
        }

        foreach ($pageAndParents as $index => $currentPageId) {
            $currentPageId = (int)$currentPageId;

            if (!isset($rulesByPageId[$currentPageId])) {
                continue;
            }

            $rule = $rulesByPageId[$currentPageId];

            /*
             * index = 0 — прямое право текущей страницы.
             * index > 0 — право родительской страницы.
             */
            $isDirectPage = $index === 0;

            /*
             * Право родителя применяется только при
             * includeChildren = true.
             */
            if (!$isDirectPage && !$rule['includeChildren']) {
                continue;
            }

            if (
                $permission === 'view'
                && ($rule['canView'] || $rule['canEdit'])
            ) {
                return true;
            }

            if (
                $permission === 'edit'
                && $rule['canEdit']
            ) {
                return true;
            }
        }

        return false;
    }

    /**
     * Получает текущую страницу и всех её родителей.
     *
     * Результат:
     * [
     *     текущая страница,
     *     родитель,
     *     родитель родителя,
     * ]
     */
    public static function getPageAndParentIds(
        int $siteId,
        int $pageId
    ): array {
        if ($siteId <= 0 || $pageId <= 0) {
            return [];
        }

        $pdo = sb_db();

        $ids = [];
        $visited = [];
        $currentPageId = $pageId;

        /*
         * Ограничение защищает от ошибочных циклов
         * в структуре страниц.
         */
        for ($i = 0; $i < 100; $i++) {
            if ($currentPageId <= 0) {
                break;
            }

            if (isset($visited[$currentPageId])) {
                break;
            }

            $visited[$currentPageId] = true;

            $stmt = $pdo->prepare("
                SELECT
                    id,
                    parent_id
                FROM sitebuilder.page
                WHERE site_id = :site_id
                  AND id = :page_id
                LIMIT 1
            ");

            $stmt->execute([
                ':site_id' => $siteId,
                ':page_id' => $currentPageId,
            ]);

            $row = $stmt->fetch(PDO::FETCH_ASSOC);

            if (!$row) {
                break;
            }

            $ids[] = (int)$row['id'];
            $currentPageId = (int)($row['parent_id'] ?? 0);
        }

        return $ids;
    }

    public static function getPageIdsWithAccess(
        int $siteId,
        string $accessCode
    ): array {
        if ($siteId <= 0) {
            return [];
        }

        $accessCode = self::normalizeAccessCode($accessCode);

        $pdo = sb_db();

        $stmt = $pdo->prepare("
            SELECT page_id
            FROM sitebuilder.page_access
            WHERE site_id = :site_id
              AND access_code = :access_code
              AND (
                    can_view = TRUE
                 OR can_edit = TRUE
              )
            ORDER BY page_id ASC
        ");

        $stmt->execute([
            ':site_id' => $siteId,
            ':access_code' => $accessCode,
        ]);

        $ids = [];

        while ($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
            $pageId = (int)($row['page_id'] ?? 0);

            if ($pageId > 0) {
                $ids[] = $pageId;
            }
        }

        return array_values(array_unique($ids));
    }

    private static function mapRow(array $row): array
    {
        return [
            'id' => (int)($row['id'] ?? 0),
            'siteId' => (int)($row['site_id'] ?? 0),
            'pageId' => (int)($row['page_id'] ?? 0),
            'accessCode' => (string)($row['access_code'] ?? ''),
            'canView' => self::boolValue($row['can_view'] ?? false),
            'canEdit' => self::boolValue($row['can_edit'] ?? false),
            'includeChildren' => self::boolValue(
                $row['include_children'] ?? false
            ),
            'createdBy' => isset($row['created_by'])
                ? (int)$row['created_by']
                : 0,
            'createdAt' => (string)($row['created_at'] ?? ''),
            'updatedAt' => (string)($row['updated_at'] ?? ''),
        ];
    }

    private static function boolValue($value): bool
    {
        return $value === true
            || $value === 1
            || $value === '1'
            || $value === 't'
            || $value === 'true'
            || $value === 'Y'
            || $value === 'y';
    }
}

У таблицы должен существовать уникальный индекс, иначе ON CONFLICT не сработает:

CREATE UNIQUE INDEX IF NOT EXISTS uq_page_access_site_page_code
ON sitebuilder.page_access (
    site_id,
    page_id,
    access_code
);

После замены файла следующим нужно обновить page_access.php, чтобы он использовал pageBelongsToSite() и проверял результат удаления.