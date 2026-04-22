Проблема уже точная:

при `site.create` у тебя в `sb_write_access()` происходит **повторная вставка одной и той же пары**:

```text
(site_id, access_code) = (6, U1)
```

А в БД на это есть уникальный индекс:

```text
uq_access_site_code
```

То есть ошибка не в PostgreSQL и не в AJAX, а в логике `sb_write_access()`.

---

# Почему это происходит

Сейчас у тебя `sb_write_access()` работает по схеме:

* если у строки есть `id > 0` → `UPDATE`
* если `id = 0` или `id` нет → `INSERT`

Но в access-данных у тебя, скорее всего, во время `site.create` формируется массив, где одна и та же запись для:

* `siteId = 6`
* `accessCode = U1`

встречается повторно или приходит как новая строка без корректного `id`.

И тогда `sb_write_access()` пытается вставить ее еще раз, а БД справедливо режет это по unique constraint.

---

# Как починить правильно

Для `access` надо опираться **не на `id`**, а на уникальную бизнес-пару:

* `site_id`
* `access_code`

То есть `sb_write_access()` нужно переписать на `UPSERT` по:

```sql
ON CONFLICT (site_id, access_code)
DO UPDATE ...
```

---

# Что сделать

## Замени функцию `sb_write_access()` в файле:

```text
/local/sitebuilder/lib/storage_db_extra.php
```

на вот эту:

```php id="h4j8d1"
function sb_write_access(array $rows): bool
{
    $pdo = sb_db();
    $pdo->beginTransaction();

    try {
        $normalizedRows = [];
        $seen = [];

        foreach ($rows as $row) {
            $siteId = (int)($row['siteId'] ?? 0);
            $accessCode = trim((string)($row['accessCode'] ?? ''));
            $role = trim((string)($row['role'] ?? ''));

            if ($siteId <= 0 || $accessCode === '' || $role === '') {
                continue;
            }

            $key = $siteId . '|' . $accessCode;

            $normalizedRows[$key] = [
                'siteId' => $siteId,
                'accessCode' => $accessCode,
                'role' => $role,
                'createdBy' => isset($row['createdBy']) ? (int)$row['createdBy'] : null,
                'createdAt' => (string)($row['createdAt'] ?? date('Y-m-d H:i:s')),
                'updatedBy' => isset($row['updatedBy']) ? (int)$row['updatedBy'] : null,
                'updatedAt' => (string)($row['updatedAt'] ?? date('Y-m-d H:i:s')),
            ];

            $seen[] = $key;
        }

        foreach ($normalizedRows as $row) {
            sb_db_execute("
                INSERT INTO sitebuilder.access (
                    site_id,
                    access_code,
                    role,
                    created_by,
                    created_at,
                    updated_by,
                    updated_at
                ) VALUES (
                    :site_id,
                    :access_code,
                    :role,
                    :created_by,
                    :created_at,
                    :updated_by,
                    :updated_at
                )
                ON CONFLICT (site_id, access_code)
                DO UPDATE SET
                    role = EXCLUDED.role,
                    created_by = EXCLUDED.created_by,
                    created_at = EXCLUDED.created_at,
                    updated_by = EXCLUDED.updated_by,
                    updated_at = EXCLUDED.updated_at
            ", [
                ':site_id' => $row['siteId'],
                ':access_code' => $row['accessCode'],
                ':role' => $row['role'],
                ':created_by' => $row['createdBy'],
                ':created_at' => $row['createdAt'],
                ':updated_by' => $row['updatedBy'],
                ':updated_at' => $row['updatedAt'],
            ]);
        }

        $existingRows = sb_db_fetch_all("
            SELECT site_id, access_code
            FROM sitebuilder.access
        ");

        $keysToKeep = array_flip(array_keys($normalizedRows));

        foreach ($existingRows as $existing) {
            $key = (int)$existing['site_id'] . '|' . (string)$existing['access_code'];

            if (!isset($keysToKeep[$key])) {
                sb_db_execute("
                    DELETE FROM sitebuilder.access
                    WHERE site_id = :site_id
                      AND access_code = :access_code
                ", [
                    ':site_id' => (int)$existing['site_id'],
                    ':access_code' => (string)$existing['access_code'],
                ]);
            }
        }

        $pdo->commit();
        return true;
    } catch (Throwable $e) {
        $pdo->rollBack();
        throw $e;
    }
}
```

---

# Что это исправит

Теперь функция:

* не будет дублировать `(site_id, access_code)`
* будет обновлять существующую запись, а не пытаться вставить ее второй раз
* будет устойчива даже если в массиве есть повторяющиеся строки

---

# Важный момент

Я специально в SQL написал таблицу со схемой:

```sql
sitebuilder.access
```

Потому что ты уже сам писал, что без схемы у тебя часть запросов ломалась.

---

# После замены

Снова попробуй создать сайт.

---

# Если после этого будет следующая ошибка

Тогда, скорее всего, аналогичная проблема может вылезти в:

* `sb_write_layouts()`
* `sb_write_menus()`

Но сначала надо закрыть `access`, потому что сейчас именно он стопорит `site.create`.

---

# Что я бы сделал следующим шагом

После того как создание сайта пройдет, я бы сразу еще проверил и укрепил:

* `sb_write_layouts()`
* `sb_write_menus()`
* `sb_write_sites()`

чтобы они тоже были не “по id любой ценой”, а устойчивыми к реальным данным.

Если хочешь, следующим сообщением я дам тебе **пачку исправленных write-функций для всех сущностей**, чтобы дальше не ловить такие unique-ошибки по одной.
