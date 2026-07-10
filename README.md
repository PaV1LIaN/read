Ошибка теперь точная:

SQLSTATE[42P10]
нет уникального ограничения, соответствующего ON CONFLICT

В PageAccessRepository.php используется:

ON CONFLICT (site_id, page_id, access_code)

Но в существующей таблице sitebuilder.page_access нет уникального индекса на эти три поля.

CREATE TABLE IF NOT EXISTS не изменяет уже существующую таблицу, поэтому ограничение не добавилось.

Создай исправляющую миграцию

Файл:

/local/sitebuilder/migrations/fix_page_access_unique.php

Полный код:

<?php

require_once $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/db.php';

global $USER;

if (!is_object($USER) || !$USER->IsAdmin()) {
    http_response_code(403);
    die('ACCESS_DENIED');
}

header('Content-Type: text/plain; charset=UTF-8');

try {
    $pdo = sb_db();

    $pdo->beginTransaction();

    /*
     * На случай, если уже появились повторяющиеся записи:
     * оставляем запись с самым большим ID.
     */
    $pdo->exec("
        DELETE FROM sitebuilder.page_access AS old_row
        USING sitebuilder.page_access AS new_row
        WHERE old_row.site_id = new_row.site_id
          AND old_row.page_id = new_row.page_id
          AND old_row.access_code = new_row.access_code
          AND old_row.id < new_row.id
    ");

    /*
     * Уникальный индекс нужен для:
     *
     * ON CONFLICT (site_id, page_id, access_code)
     */
    $pdo->exec("
        CREATE UNIQUE INDEX IF NOT EXISTS uq_page_access_site_page_code
        ON sitebuilder.page_access (
            site_id,
            page_id,
            access_code
        )
    ");

    $pdo->commit();

    echo "OK: unique index created\n\n";

    $stmt = $pdo->query("
        SELECT
            indexname,
            indexdef
        FROM pg_indexes
        WHERE schemaname = 'sitebuilder'
          AND tablename = 'page_access'
        ORDER BY indexname
    ");

    $indexes = $stmt->fetchAll(PDO::FETCH_ASSOC);

    print_r($indexes);
} catch (Throwable $e) {
    if (isset($pdo) && $pdo->inTransaction()) {
        $pdo->rollBack();
    }

    http_response_code(500);

    echo "ERROR:\n";
    echo $e->getMessage() . "\n";
    echo $e->getFile() . ':' . $e->getLine();
}

Открой:

https://portal24.itsnn.ru/local/sitebuilder/migrations/fix_page_access_unique.php

Должен появиться индекс примерно такого вида:

[indexname] => uq_page_access_site_page_code

и определение:

CREATE UNIQUE INDEX uq_page_access_site_page_code
ON sitebuilder.page_access
USING btree (site_id, page_id, access_code)

После этого снова выполни тот же консольный тест pageAccess.save.

Ожидаемый результат:

3. HTTP pageAccess.save: 200
УСПЕШНО: право сохранено

После успешного запуска удали временный файл:

/local/sitebuilder/migrations/fix_page_access_unique.php