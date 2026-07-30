Обработка заняла 2m 13s

Отлично — значит сама SQL-миграция этапа 5 рабочая. Ошибка возникает именно при запуске через PHP: либо при соединении PHP с базой, либо при регистрации миграции в schema_migration. Обычная страница скрывает настоящую причину за MIGRATION_FAILED.

Я подготовил временную диагностическую страницу:

Скачать migration_debug.php

Что сделать

1. Загрузи файл сюда:



/local/sitebuilder/tools/migration_debug.php

2. Открой под администратором Битрикс:



https://portal24.itsnn.ru/local/sitebuilder/tools/migration_debug.php

3. В верхнем блоке проверь подключение PHP:



is_replica = false
transaction_read_only = off

4. Нажми:



Запустить миграцию и показать точную ошибку

Страница покажет полную цепочку:

Класс
Message
Code
SQLSTATE
Driver message
Details

Пришли скриншот этого блока или скопируй значения Message, SQLSTATE и Driver message.

После диагностики файл обязательно удали:

/local/sitebuilder/tools/migration_debug.php

Сейчас обычную страницу apply_stage13_migration.php повторно не запускай. Базу восстанавливать не требуется.

<?php

declare(strict_types=1);

define('NO_KEEP_STATISTIC', true);
define('NO_AGENT_STATISTIC', true);
define('NOT_CHECK_PERMISSIONS', true);

require_once $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/auth.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/db.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/MigrationService.php';

sitebuilder_require_bitrix_admin();

global $USER;

header('Cache-Control: no-store, no-cache, must-revalidate, max-age=0');
header('Pragma: no-cache');

function sbMigrationDebugEscape(string $value): string
{
    return htmlspecialchars($value, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
}

function sbMigrationDebugThrowable(Throwable $e): array
{
    $items = [];
    $current = $e;
    $depth = 0;

    while ($current instanceof Throwable && $depth < 10) {
        $item = [
            'depth' => $depth,
            'class' => get_class($current),
            'message' => $current->getMessage(),
            'code' => (string)$current->getCode(),
            'sqlstate' => '',
            'driverCode' => '',
            'driverMessage' => '',
            'details' => [],
        ];

        if ($current instanceof PDOException) {
            $errorInfo = $current->errorInfo ?? null;
            if (is_array($errorInfo)) {
                $item['sqlstate'] = isset($errorInfo[0]) ? (string)$errorInfo[0] : '';
                $item['driverCode'] = isset($errorInfo[1]) ? (string)$errorInfo[1] : '';
                $item['driverMessage'] = isset($errorInfo[2]) ? (string)$errorInfo[2] : '';
            }
        }

        if (method_exists($current, 'details')) {
            try {
                $details = $current->details();
                $item['details'] = is_array($details) ? $details : [];
            } catch (Throwable $ignored) {
                $item['details'] = [];
            }
        }

        $items[] = $item;
        $current = $current->getPrevious();
        $depth++;
    }

    return $items;
}

$dbInfo = [];
$dbInfoError = '';
$result = null;
$errorChain = [];

try {
    $dbInfo = sb_db_fetch_one(
        "SELECT
            current_database() AS database_name,
            current_user AS database_user,
            inet_server_addr()::text AS server_ip,
            inet_server_port() AS server_port,
            pg_is_in_recovery() AS is_replica,
            current_setting('transaction_read_only') AS transaction_read_only,
            current_setting('default_transaction_read_only') AS default_transaction_read_only"
    ) ?? [];
} catch (Throwable $e) {
    $dbInfoError = $e->getMessage();
}

if (($_SERVER['REQUEST_METHOD'] ?? 'GET') === 'POST') {
    if (!check_bitrix_sessid()) {
        $errorChain[] = [
            'depth' => 0,
            'class' => 'CSRF',
            'message' => 'Сессия устарела. Обновите страницу.',
            'code' => '',
            'sqlstate' => '',
            'driverCode' => '',
            'driverMessage' => '',
            'details' => [],
        ];
    } else {
        try {
            $result = MigrationService::bootstrap((int)$USER->GetID());
        } catch (Throwable $e) {
            $errorChain = sbMigrationDebugThrowable($e);
        }
    }
}

?>
<!doctype html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Диагностика миграций SiteBuilder</title>
    <style>
        body{margin:0;padding:24px;font-family:Arial,sans-serif;background:#f3f6fb;color:#1f2937}
        .card{max-width:1100px;margin:0 auto;padding:24px;background:#fff;border:1px solid #e5e7eb;border-radius:14px}
        table{width:100%;border-collapse:collapse;margin:16px 0}th,td{padding:9px;border:1px solid #e5e7eb;text-align:left;vertical-align:top;font-size:14px}
        pre{white-space:pre-wrap;word-break:break-word;background:#111827;color:#f9fafb;padding:14px;border-radius:10px}
        button{padding:11px 16px;border:0;border-radius:8px;background:#dc2626;color:#fff;font-weight:700;cursor:pointer}
        .ok{padding:12px;background:#f0fdf4;border:1px solid #bbf7d0;color:#166534;border-radius:8px}
        .warn{padding:12px;background:#fff7ed;border:1px solid #fed7aa;color:#9a3412;border-radius:8px}
        code{background:#f3f4f6;padding:2px 5px;border-radius:4px}
    </style>
</head>
<body>
<div class="card">
    <h1>Диагностика миграций SiteBuilder</h1>
    <p class="warn"><strong>Временный служебный файл.</strong> Доступен только администратору Битрикс. После получения ошибки удали его с сервера.</p>

    <h2>Подключение PHP к PostgreSQL</h2>
    <?php if ($dbInfoError !== ''): ?>
        <pre><?= sbMigrationDebugEscape($dbInfoError) ?></pre>
    <?php else: ?>
        <table>
            <?php foreach ($dbInfo as $key => $value): ?>
                <tr>
                    <th><?= sbMigrationDebugEscape((string)$key) ?></th>
                    <td><?= sbMigrationDebugEscape(is_bool($value) ? ($value ? 'true' : 'false') : (string)$value) ?></td>
                </tr>
            <?php endforeach; ?>
        </table>
    <?php endif; ?>

    <?php if (is_array($result)): ?>
        <div class="ok">Миграции выполнены успешно.</div>
        <pre><?= sbMigrationDebugEscape((string)json_encode($result, JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES | JSON_PRETTY_PRINT)) ?></pre>
    <?php endif; ?>

    <?php if ($errorChain !== []): ?>
        <h2>Цепочка исключений</h2>
        <?php foreach ($errorChain as $item): ?>
            <table>
                <tr><th>Уровень</th><td><?= (int)$item['depth'] ?></td></tr>
                <tr><th>Класс</th><td><code><?= sbMigrationDebugEscape((string)$item['class']) ?></code></td></tr>
                <tr><th>Message</th><td><?= sbMigrationDebugEscape((string)$item['message']) ?></td></tr>
                <tr><th>Code</th><td><?= sbMigrationDebugEscape((string)$item['code']) ?></td></tr>
                <tr><th>SQLSTATE</th><td><?= sbMigrationDebugEscape((string)$item['sqlstate']) ?></td></tr>
                <tr><th>Driver code</th><td><?= sbMigrationDebugEscape((string)$item['driverCode']) ?></td></tr>
                <tr><th>Driver message</th><td><?= sbMigrationDebugEscape((string)$item['driverMessage']) ?></td></tr>
                <tr><th>Details</th><td><pre><?= sbMigrationDebugEscape((string)json_encode($item['details'], JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES | JSON_PRETTY_PRINT)) ?></pre></td></tr>
            </table>
        <?php endforeach; ?>
    <?php endif; ?>

    <form method="post">
        <?= bitrix_sessid_post() ?>
        <button type="submit">Запустить миграцию и показать точную ошибку</button>
    </form>
</div>
</body>
</html