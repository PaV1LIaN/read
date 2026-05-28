Отлично. Дальше добавим поиск по пользователям.

Сейчас есть:

/local/mvc_demo/admin/users
/local/mvc_demo/admin/users?page=2

Сделаем так:

/local/mvc_demo/admin/users?q=ivan
/local/mvc_demo/admin/users?q=ivan&page=2

То есть пользователь сможет искать по:

ID
логину
имени
фамилии
email


---

1. Обнови /local/mvc_demo/Models/User.php

Внутрь класса User добавь два метода:

public static function countSearch(string $search = ''): int
{
    $search = trim($search);

    if ($search === '') {
        return self::count();
    }

    return (int)Db::value("
        SELECT COUNT(*)
        FROM b_user
        WHERE
            CAST(ID AS CHAR) LIKE :q
            OR LOGIN LIKE :q
            OR NAME LIKE :q
            OR LAST_NAME LIKE :q
            OR SECOND_NAME LIKE :q
            OR EMAIL LIKE :q
    ", [
        'q' => '%' . $search . '%',
    ]);
}

public static function searchPage(string $search = '', int $limit = 10, int $offset = 0): array
{
    $search = trim($search);
    $limit = max(1, min($limit, 100));
    $offset = max(0, $offset);

    if ($search === '') {
        return self::latestPage($limit, $offset);
    }

    return Db::fetchAll("
        SELECT
            ID,
            LOGIN,
            NAME,
            LAST_NAME,
            SECOND_NAME,
            EMAIL,
            ACTIVE,
            DATE_REGISTER,
            LAST_LOGIN
        FROM b_user
        WHERE
            CAST(ID AS CHAR) LIKE :q
            OR LOGIN LIKE :q
            OR NAME LIKE :q
            OR LAST_NAME LIKE :q
            OR SECOND_NAME LIKE :q
            OR EMAIL LIKE :q
        ORDER BY ID DESC
        LIMIT {$limit} OFFSET {$offset}
    ", [
        'q' => '%' . $search . '%',
    ]);
}

Что это значит простыми словами:

countSearch() — считает, сколько найдено пользователей.
searchPage()  — получает только нужный кусок найденных пользователей.


---

2. Обнови /local/mvc_demo/Services/UserService.php

Найди метод:

public function paginateForTable(int $page = 1, int $perPage = 10): array

И замени его на:

public function paginateForTable(int $page = 1, int $perPage = 10, string $search = ''): array
{
    $search = trim($search);

    $total = User::countSearch($search);

    $paginator = new Paginator($total, $page, $perPage);

    $users = User::searchPage(
        $search,
        $paginator->perPage(),
        $paginator->offset()
    );

    $items = [];

    foreach ($users as $user) {
        $items[] = $this->prepareForTable($user);
    }

    return [
        'items' => $items,
        'pagination' => $paginator->toArray(),
        'search' => $search,
    ];
}

Теперь сервис умеет не только страницы, но и поиск.


---

3. Обнови метод users() в /local/mvc_demo/Controllers/AdminController.php

Замени метод users() на этот:

public function users(): Response
{
    $page = (int)$this->request->get('page', 1);
    $search = trim((string)$this->request->get('q', ''));

    $userService = new UserService();

    $result = $userService->paginateForTable($page, 10, $search);

    return $this->render('admin/users', [
        'title' => 'Пользователи',
        'users' => $result['items'],
        'pagination' => $result['pagination'],
        'search' => $result['search'],
    ]);
}

Что теперь делает контроллер:

1. Берёт page из URL.
2. Берёт q из URL.
3. Передаёт всё в UserService.
4. Отдаёт users, pagination и search во View.


---

4. Замени /local/mvc_demo/Views/admin/users.php

Полностью замени файл:

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

$pageUrl = '/local/mvc_demo/admin/users';
$search = trim((string)($search ?? ''));

$makePageUrl = static function (int $page) use ($pageUrl, $search): string {
    $params = [
        'page' => $page,
    ];

    if ($search !== '') {
        $params['q'] = $search;
    }

    return $pageUrl . '?' . http_build_query($params);
};

?>

<div class="mvc-card">
    <h1 class="mvc-page-title">
        <?= htmlspecialcharsbx($title ?? 'Пользователи') ?>
    </h1>

    <p class="mvc-page-text">
        Это список пользователей из таблицы <span class="mvc-code">b_user</span> с поиском и пагинацией.
    </p>

    <form method="get" action="<?= htmlspecialcharsbx($pageUrl) ?>" style="margin-top: 24px; display: flex; gap: 10px; align-items: center;">
        <input
            type="text"
            name="q"
            value="<?= htmlspecialcharsbx($search) ?>"
            placeholder="Поиск по ID, логину, ФИО или email"
            style="flex: 1; min-height: 42px; padding: 8px 12px; border: 1px solid #d1d5db; border-radius: 10px;"
        >

        <button
            type="submit"
            style="min-height: 42px; padding: 0 18px; border: 0; border-radius: 10px; background: #2563eb; color: #fff; font-weight: 600; cursor: pointer;"
        >
            Найти
        </button>

        <?php if ($search !== ''): ?>
            <a
                href="<?= htmlspecialcharsbx($pageUrl) ?>"
                style="min-height: 42px; display: inline-flex; align-items: center; padding: 0 14px; border: 1px solid #d1d5db; border-radius: 10px; text-decoration: none;"
            >
                Сбросить
            </a>
        <?php endif; ?>
    </form>

    <?php if ($search !== ''): ?>
        <div class="mvc-info">
            Поиск:
            <span class="mvc-code"><?= htmlspecialcharsbx($search) ?></span>
        </div>
    <?php endif; ?>

    <?php if (!empty($pagination)): ?>
        <div class="mvc-info">
            Показаны записи
            <span class="mvc-code"><?= htmlspecialcharsbx($pagination['from'] ?? 0) ?></span>
            —
            <span class="mvc-code"><?= htmlspecialcharsbx($pagination['to'] ?? 0) ?></span>
            из
            <span class="mvc-code"><?= htmlspecialcharsbx($pagination['total'] ?? 0) ?></span>
        </div>
    <?php endif; ?>

    <div class="mvc-info">
        <?php if (empty($users)): ?>
            <p style="margin: 0;">
                Пользователи не найдены.
            </p>
        <?php else: ?>
            <table style="width: 100%; border-collapse: collapse;">
                <thead>
                    <tr>
                        <th style="text-align: left; padding: 8px; border-bottom: 1px solid #e5e7eb;">ID</th>
                        <th style="text-align: left; padding: 8px; border-bottom: 1px solid #e5e7eb;">Логин</th>
                        <th style="text-align: left; padding: 8px; border-bottom: 1px solid #e5e7eb;">ФИО</th>
                        <th style="text-align: left; padding: 8px; border-bottom: 1px solid #e5e7eb;">Email</th>
                        <th style="text-align: left; padding: 8px; border-bottom: 1px solid #e5e7eb;">Активен</th>
                        <th style="text-align: left; padding: 8px; border-bottom: 1px solid #e5e7eb;">Действие</th>
                    </tr>
                </thead>

                <tbody>
                    <?php foreach (($users ?? []) as $user): ?>
                        <tr>
                            <td style="padding: 8px; border-bottom: 1px solid #e5e7eb;">
                                <?= htmlspecialcharsbx($user['id'] ?? '') ?>
                            </td>

                            <td style="padding: 8px; border-bottom: 1px solid #e5e7eb;">
                                <?= htmlspecialcharsbx($user['login'] ?? '') ?>
                            </td>

                            <td style="padding: 8px; border-bottom: 1px solid #e5e7eb;">
                                <?= htmlspecialcharsbx($user['full_name'] ?? '') ?>
                            </td>

                            <td style="padding: 8px; border-bottom: 1px solid #e5e7eb;">
                                <?= htmlspecialcharsbx($user['email'] ?? '') ?>
                            </td>

                            <td style="padding: 8px; border-bottom: 1px solid #e5e7eb;">
                                <?= htmlspecialcharsbx($user['active_label'] ?? '') ?>
                            </td>

                            <td style="padding: 8px; border-bottom: 1px solid #e5e7eb;">
                                <a href="/local/mvc_demo/admin/users/<?= (int)($user['id'] ?? 0) ?>">
                                    Открыть
                                </a>
                            </td>
                        </tr>
                    <?php endforeach; ?>
                </tbody>
            </table>
        <?php endif; ?>
    </div>

    <?php if (!empty($pagination) && (int)($pagination['total_pages'] ?? 1) > 1): ?>
        <div style="margin-top: 20px; display: flex; gap: 8px; align-items: center; flex-wrap: wrap;">
            <?php if (!empty($pagination['has_prev'])): ?>
                <a
                    href="<?= htmlspecialcharsbx($makePageUrl((int)$pagination['prev_page'])) ?>"
                    style="padding: 8px 12px; border: 1px solid #d1d5db; border-radius: 8px; text-decoration: none;"
                >
                    ← Назад
                </a>
            <?php endif; ?>

            <?php foreach (($pagination['pages'] ?? []) as $page): ?>
                <?php $isCurrent = ((int)$page === (int)($pagination['page'] ?? 1)); ?>

                <a
                    href="<?= htmlspecialcharsbx($makePageUrl((int)$page)) ?>"
                    style="
                        padding: 8px 12px;
                        border: 1px solid <?= $isCurrent ? '#2563eb' : '#d1d5db' ?>;
                        border-radius: 8px;
                        text-decoration: none;
                        background: <?= $isCurrent ? '#2563eb' : '#fff' ?>;
                        color: <?= $isCurrent ? '#fff' : '#111827' ?>;
                    "
                >
                    <?= (int)$page ?>
                </a>
            <?php endforeach; ?>

            <?php if (!empty($pagination['has_next'])): ?>
                <a
                    href="<?= htmlspecialcharsbx($makePageUrl((int)$pagination['next_page'])) ?>"
                    style="padding: 8px 12px; border: 1px solid #d1d5db; border-radius: 8px; text-decoration: none;"
                >
                    Вперёд →
                </a>
            <?php endif; ?>
        </div>
    <?php endif; ?>
</div>


---

5. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/admin/users

Проверь поиск:

https://bitrix24-stage.gaz.ru/local/mvc_demo/admin/users?q=admin

Проверь вторую страницу поиска:

https://bitrix24-stage.gaz.ru/local/mvc_demo/admin/users?q=admin&page=2


---

Что мы сделали

Теперь цепочка стала такой:

URL содержит q
  ↓
Controller берёт q из Request
  ↓
Service передаёт q в Model
  ↓
Model делает WHERE
  ↓
Paginator считает страницы только по найденным записям
  ↓
View показывает поиск, таблицу и страницы

Главная мысль:

page отвечает за номер страницы
q отвечает за строку поиска

Дальше логичный шаг — сделать универсальный Table/List компонент для вывода таблиц, чтобы не писать HTML-таблицу руками каждый раз.