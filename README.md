Да, тогда не нужно создавать новое подключение вручную.

Надо просто переделать `db.php`, чтобы он использовал твой существующий `pg_master.php` и `getPDO()`.

---

# Что сделать

## Замени `/local/sitebuilder/lib/db.php` на это

```php id="0hziw8"
<?php

function sb_db(): PDO
{
    static $pdo = null;

    if ($pdo instanceof PDO) {
        return $pdo;
    }

    require_once $_SERVER['DOCUMENT_ROOT'] . '/local/php_interface/lib/pg_master.php';

    if (!function_exists('getPDO')) {
        throw new RuntimeException('FUNCTION_getPDO_NOT_FOUND');
    }

    $pdo = getPDO();

    if (!$pdo instanceof PDO) {
        throw new RuntimeException('getPDO_DID_NOT_RETURN_PDO');
    }

    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    $pdo->setAttribute(PDO::ATTR_DEFAULT_FETCH_MODE, PDO::FETCH_ASSOC);

    try {
        $pdo->exec("SET search_path TO sitebuilder, public");
    } catch (Throwable $e) {
        // Если схема уже задана на уровне подключения — не критично.
    }

    return $pdo;
}

function sb_db_fetch_all(string $sql, array $params = []): array
{
    $stmt = sb_db()->prepare($sql);
    $stmt->execute($params);
    return $stmt->fetchAll();
}

function sb_db_fetch_one(string $sql, array $params = []): ?array
{
    $stmt = sb_db()->prepare($sql);
    $stmt->execute($params);
    $row = $stmt->fetch();

    return $row !== false ? $row : null;
}

function sb_db_execute(string $sql, array $params = []): bool
{
    $stmt = sb_db()->prepare($sql);
    return $stmt->execute($params);
}

function sb_db_last_insert_id(?string $sequence = null): int
{
    return (int)sb_db()->lastInsertId($sequence);
}

function sb_json_decode_assoc($value): array
{
    if (is_array($value)) {
        return $value;
    }

    if ($value === null || $value === '') {
        return [];
    }

    $decoded = json_decode((string)$value, true);
    return is_array($decoded) ? $decoded : [];
}
```

---

# После этого

Снова запусти:

```text id="5qz664"
/local/sitebuilder/tools/migrate_json_to_pg.php
```

---

# Если будет следующая ошибка

Тогда пришли текст ошибки, но наиболее вероятно дальше будет одно из двух:

## 1. Нет схемы `sitebuilder`

Тогда будет ошибка про relation/schema.

## 2. В `pg_master.php` `getPDO()` называется иначе

Но ты написал, что функция именно `getPDO()`, так что должно завестись.

---

# Что еще советую быстро проверить

Если хочешь убедиться, что `getPDO()` реально работает, сделай временный тест:

## `/local/sitebuilder/tools/test_pg.php`

```php id="vq113l"
<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/php_interface/lib/pg_master.php';

echo '<pre>';

if (!function_exists('getPDO')) {
    echo "getPDO not found\n";
    exit;
}

$pdo = getPDO();

if (!$pdo instanceof PDO) {
    echo "getPDO did not return PDO\n";
    exit;
}

echo "PDO OK\n";

$stmt = $pdo->query("select current_database() as db, current_schema() as schema");
$row = $stmt->fetch(PDO::FETCH_ASSOC);

print_r($row);

echo '</pre>';
```

Если он покажет `PDO OK`, значит можно спокойно продолжать миграцию.

Если хочешь, следующим сообщением я дам тебе **исправленный migration script**, в котором еще сразу добавлю `TRUNCATE/REIMPORT` режим, чтобы удобно перегонять JSON в PostgreSQL повторно.
