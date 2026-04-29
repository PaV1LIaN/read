Да, следующий этап по правам — сделать выдачу роли **не по ID пользователя**, а через поиск пользователя.

Сейчас неудобно:

```text
ID пользователя: 99
Роль: EDITOR
```

Сделаем так:

```text
Поиск пользователя: Иванов
Выбрать пользователя из списка
Роль: EDITOR
Выдать роль
```

---

## 1. Добавь action в `/local/sitebuilder/api/index.php`

В роутер добавь блок перед `UNKNOWN_ACTION`:

```php
if (
    $action === 'user.search'
) {
    require __DIR__ . '/handlers/user.php';
    exit;
}
```

---

## 2. Создай файл `/local/sitebuilder/api/handlers/user.php`

```php
<?php

global $USER;

if (!function_exists('sb_user_search_require_owner')) {
    function sb_user_search_require_owner(int $siteId): void
    {
        global $USER;

        if ($USER && $USER->IsAdmin()) {
            return;
        }

        sb_require_owner($siteId);
    }
}

if (!function_exists('sb_user_search_normalize')) {
    function sb_user_search_normalize(array $row): array
    {
        $id = (int)($row['ID'] ?? 0);

        $fio = trim(
            (string)($row['LAST_NAME'] ?? '') . ' ' .
            (string)($row['NAME'] ?? '') . ' ' .
            (string)($row['SECOND_NAME'] ?? '')
        );

        $login = (string)($row['LOGIN'] ?? '');
        $email = (string)($row['EMAIL'] ?? '');

        $title = $fio !== '' ? $fio : ($login !== '' ? $login : ('Пользователь #' . $id));

        return [
            'id' => $id,
            'name' => $fio,
            'login' => $login,
            'email' => $email,
            'title' => $title,
            'active' => (string)($row['ACTIVE'] ?? ''),
        ];
    }
}

if (!function_exists('sb_user_search_add_result')) {
    function sb_user_search_add_result(array &$results, array $row): void
    {
        $id = (int)($row['ID'] ?? 0);

        if ($id <= 0) {
            return;
        }

        $results[$id] = sb_user_search_normalize($row);
    }
}

if (!function_exists('sb_user_search_by_filter')) {
    function sb_user_search_by_filter(array $filter, array &$results, int $limit): void
    {
        $by = 'last_name';
        $order = 'asc';

        $rs = CUser::GetList(
            $by,
            $order,
            $filter,
            [
                'FIELDS' => [
                    'ID',
                    'LOGIN',
                    'EMAIL',
                    'NAME',
                    'LAST_NAME',
                    'SECOND_NAME',
                    'ACTIVE',
                ],
            ]
        );

        while ($row = $rs->Fetch()) {
            sb_user_search_add_result($results, $row);

            if (count($results) >= $limit) {
                break;
            }
        }
    }
}

if ($action === 'user.search') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $query = trim((string)($_POST['query'] ?? ''));
    $limit = (int)($_POST['limit'] ?? 10);

    $limit = max(1, min(20, $limit));

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    sb_user_search_require_owner($siteId);

    if ($query === '') {
        sb_json_ok([
            'users' => [],
            'handler' => 'user',
            'action' => 'user.search',
            'file' => __FILE__,
        ]);
    }

    if (!preg_match('/^\d+$/', $query) && mb_strlen($query) < 2) {
        sb_json_error('QUERY_TOO_SHORT', 422);
    }

    $results = [];

    if (preg_match('/^\d+$/', $query)) {
        $rs = CUser::GetByID((int)$query);
        $row = $rs ? $rs->Fetch() : null;

        if ($row) {
            sb_user_search_add_result($results, $row);
        }
    }

    if (count($results) < $limit) {
        sb_user_search_by_filter([
            'ACTIVE' => 'Y',
            '%LOGIN' => $query,
        ], $results, $limit);
    }

    if (count($results) < $limit) {
        sb_user_search_by_filter([
            'ACTIVE' => 'Y',
            '%EMAIL' => $query,
        ], $results, $limit);
    }

    if (count($results) < $limit) {
        sb_user_search_by_filter([
            'ACTIVE' => 'Y',
            '%NAME' => $query,
        ], $results, $limit);
    }

    if (count($results) < $limit) {
        sb_user_search_by_filter([
            'ACTIVE' => 'Y',
            '%LAST_NAME' => $query,
        ], $results, $limit);
    }

    sb_json_ok([
        'users' => array_values($results),
        'handler' => 'user',
        'action' => 'user.search',
        'file' => __FILE__,
    ]);
}

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'user',
    'action' => $action,
    'file' => __FILE__,
]);
```

---

## 3. Быстрая проверка через консоль

```js
fetch('/local/sitebuilder/api.php', {
  method: 'POST',
  headers: {'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'},
  body: new URLSearchParams({
    action: 'user.search',
    siteId: '11',
    query: '99',
    sessid: BX.bitrix_sessid()
  }),
  credentials: 'same-origin'
})
.then(r => r.json())
.then(console.log);
```

Ожидаемо:

```json
{
  "ok": true,
  "users": [
    {
      "id": 99,
      "title": "...",
      "login": "...",
      "email": "..."
    }
  ]
}
```

---

## 4. После проверки

Дальше я дам обновление `editor.php`, где блок **«Права пользователей»** будет уже с поиском:

```text
Поиск пользователя
[Иванов__________]
↓
Иванов Иван — ivanov@example.ru
[Выбрать]

Роль
[EDITOR]

[Выдать роль]
```

И отдельно уберём лишние админские блоки для обычного `EDITOR`: группа Битрикс24, права пользователей, ответ API и удаление сайта.
