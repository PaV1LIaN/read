"ProjectsDB"	"bx_user"	"192.168.7.100/32"	5432	true	"on"	"off"

<?php
const DB_USER = 'bx_user';
const DB_PASS = '25##PostPassBX';

const DB_NODES = [
    "pgsql:host=192.168.7.101;port=5432;dbname=ProjectsDB",
    "pgsql:host=192.168.7.102;port=5432;dbname=ProjectsDB",
    "pgsql:host=192.168.7.100;port=5432;dbname=ProjectsDB",
];

const DB_MASTER_CACHE_TTL = 5;

function cacheGet(string $key): ?string
{
    if (function_exists('apcu_fetch')) {
        $ok = false;
        $val = apcu_fetch($key, $ok);
        return $ok ? (string)$val : null;
    }
    return null;
}

function cacheSet(string $key, string $value, int $ttl): void
{
    if (function_exists('apcu_store')) {
        apcu_store($key, $value, $ttl);
    }
}

function makePdo(string $dsn): PDO
{
    if (stripos($dsn, 'pgsql:') !== 0) {
        $dsn = 'pgsql:' . $dsn;
    }

    return new PDO(
        $dsn,
        DB_USER,
        DB_PASS,
        [
            PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
            PDO::ATTR_EMULATE_PREPARES   => false,
        ]
    );
}

function isMaster(PDO $pdo): bool
{
    $inRecovery = $pdo->query("SELECT pg_is_in_recovery()")->fetchColumn();
    return !((string)$inRecovery === 't' || (string)$inRecovery === '1' || $inRecovery === true || $inRecovery === 1);
}

function findMasterDsn(): string
{
    $cacheKey = 'db_master_dsn';
    $cached = cacheGet($cacheKey);
    if ($cached) {
        return $cached;
    }

    $nodes = DB_NODES;
    shuffle($nodes);

    $diag = [];

    foreach ($nodes as $dsn) {
        try {
            $pdo = makePdo($dsn);
            $rec = $pdo->query("SELECT pg_is_in_recovery()")->fetchColumn();
            $diag[] = $dsn . " pg_is_in_recovery=" . (string)$rec;

            if (isMaster($pdo)) {
                cacheSet($cacheKey, $dsn, DB_MASTER_CACHE_TTL);
                return $dsn;
            }
        } catch (Throwable $e) {
            $diag[] = $dsn . " ERROR=" . $e->getMessage();
        }
    }

    error_log("MASTER NOT FOUND. DIAG: " . implode(" | ", $diag));
	throw new RuntimeException('DB_UNAVAILABLE');
}

function getPdo(): PDO
{
    static $pdo = null;
    static $dsnUsed = null;

    $dsn = findMasterDsn();

    if ($pdo instanceof PDO && $dsnUsed === $dsn) {
        return $pdo;
    }

    $pdo = makePdo($dsn);
    $dsnUsed = $dsn;

    if (!isMaster($pdo)) {
        cacheSet('db_master_dsn', '', 1);
        $dsn2 = findMasterDsn();
        $pdo2 = makePdo($dsn2);

        if (!isMaster($pdo2)) {
            throw new RuntimeException('Мастер не найден');
        }

        $pdo = $pdo2;
        $dsnUsed = $dsn2;
    }

    return $pdo;
}
