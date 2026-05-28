Идём дальше. Сейчас сделаем пагинацию.

Что такое пагинация

Сейчас страница пользователей показывает последние 10 пользователей:

/admin/users

Но если пользователей 5000, нельзя выводить всех сразу.

Нужно так:

/admin/users?page=1
/admin/users?page=2
/admin/users?page=3

То есть страница будет показывать кусками:

1 страница — пользователи 1–10
2 страница — пользователи 11–20
3 страница — пользователи 21–30


---

1. Создаём /local/mvc/Core/Paginator.php

<?php

namespace Local\Mvc\Core;

/**
 * Paginator
 *
 * Помощник для постраничного вывода.
 */
class Paginator
{
    private int $total;
    private int $page;
    private int $perPage;

    public function __construct(int $total, int $page = 1, int $perPage = 10)
    {
        $this->total = max(0, $total);
        $this->perPage = max(1, min($perPage, 100));

        $totalPages = $this->totalPages();

        $page = max(1, $page);

        if ($totalPages > 0) {
            $page = min($page, $totalPages);
        }

        $this->page = $page;
    }

    public function total(): int
    {
        return $this->total;
    }

    public function page(): int
    {
        return $this->page;
    }

    public function perPage(): int
    {
        return $this->perPage;
    }

    public function totalPages(): int
    {
        if ($this->total === 0) {
            return 1;
        }

        return (int)ceil($this->total / $this->perPage);
    }

    public function offset(): int
    {
        return ($this->page - 1) * $this->perPage;
    }

    public function hasPrev(): bool
    {
        return $this->page > 1;
    }

    public function hasNext(): bool
    {
        return $this->page < $this->totalPages();
    }

    public function prevPage(): int
    {
        return max(1, $this->page - 1);
    }

    public function nextPage(): int
    {
        return min($this->totalPages(), $this->page + 1);
    }

    public function pages(): array
    {
        $pages = [];

        $start = max(1, $this->page - 2);
        $end = min($this->totalPages(), $this->page + 2);

        for ($i = $start; $i <= $end; $i++) {
            $pages[] = $i;
        }

        return $pages;
    }

    public function from(): int
    {
        if ($this->total === 0) {
            return 0;
        }

        return $this->offset() + 1;
    }

    public function to(): int
    {
        return min($this->offset() + $this->perPage, $this->total);
    }

    public function toArray(): array
    {
        return [
            'total' => $this->total(),
            'page' => $this->page(),
            'per_page' => $this->perPage(),
            'total_pages' => $this->totalPages(),
            'offset' => $this->offset(),
            'has_prev' => $this->hasPrev(),
            'has_next' => $this->hasNext(),
            'prev_page' => $this->prevPage(),
            'next_page' => $this->nextPage(),
            'pages' => $this->pages(),
            'from' => $this->from(),
            'to' => $this->to(),
        ];
    }
}


---

2. Обновляем /local/mvc_demo/Models/User.php

Внутрь класса User добавь метод:

public static function latestPage(int $limit = 10, int $offset = 0): array
{
    $limit = max(1, min($limit, 100));
    $offset = max(0, $offset);

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
        ORDER BY ID DESC
        LIMIT {$limit} OFFSET {$offset}
    ");
}

То есть в модели теперь будет:

User::count();
User::latestPage($limit, $offset);


---

3. Обновляем /local/mvc_demo/Services/UserService.php

Замени файл полностью:

<?php

namespace Local\MvcDemo\Services;

use Local\Mvc\Core\Paginator;
use Local\MvcDemo\Models\User;

/**
 * UserService
 *
 * Готовит данные пользователей для контроллеров.
 */
class UserService
{
    public function dashboardStats(): array
    {
        return [
            'users_count' => User::count(),
        ];
    }

    public function latestForTable(int $limit = 10): array
    {
        $users = User::latest($limit);

        $result = [];

        foreach ($users as $user) {
            $result[] = $this->prepareForTable($user);
        }

        return $result;
    }

    public function paginateForTable(int $page = 1, int $perPage = 10): array
    {
        $total = User::count();

        $paginator = new Paginator($total, $page, $perPage);

        $users = User::latestPage(
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
        ];
    }

    public function findForDetail(int $id): ?array
    {
        $user = User::findForAdmin($id);

        if (!$user) {
            return null;
        }

        return $this->prepareForDetail($user);
    }

    private function prepareForTable(array $user): array
    {
        return [
            'id' => (int)($user['ID'] ?? 0),
            'login' => (string)($user['LOGIN'] ?? ''),
            'full_name' => User::fullName($user),
            'email' => (string)($user['EMAIL'] ?? ''),
            'active' => (string)($user['ACTIVE'] ?? ''),
            'active_label' => (($user['ACTIVE'] ?? '') === 'Y') ? 'Да' : 'Нет',
        ];
    }

    private function prepareForDetail(array $user): array
    {
        return [
            'id' => (int)($user['ID'] ?? 0),
            'login' => (string)($user['LOGIN'] ?? ''),
            'full_name' => User::fullName($user),
            'email' => (string)($user['EMAIL'] ?? ''),
            'active' => (string)($user['ACTIVE'] ?? ''),
            'active_label' => (($user['ACTIVE'] ?? '') === 'Y') ? 'Да' : 'Нет',
            'date_register' => (string)($user['DATE_REGISTER'] ?? ''),
            'last_login' => (string)($user['LAST_LOGIN'] ?? ''),
        ];
    }
}


---

4. Обновляем /local/mvc_demo/Controllers/AdminController.php

В методе users() было примерно так:

public function users(): Response
{
    $userService = new UserService();

    return $this->render('admin/users', [
        'title' => 'Пользователи',
        'users' => $userService->latestForTable(10),
    ]);
}

Замени метод users() на:

public function users(): Response
{
    $page = (int)$this->request->get('page', 1);

    $userService = new UserService();

    $result = $userService->paginateForTable($page, 10);

    return $this->render('admin/users', [
        'title' => 'Пользователи',
        'users' => $result['items'],
        'pagination' => $result['pagination'],
    ]);
}


---

5. Обновляем /local/mvc_demo/Views/admin/users.php

Замени файл полностью:

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

$pageUrl = '/local/mvc_demo/admin/users';

?>

<div class="mvc-card">
    <h1 class="mvc-page-title">
        <?= htmlspecialcharsbx($title ?? 'Пользователи') ?>
    </h1>

    <p class="mvc-page-text">
        Это список пользователей из таблицы <span class="mvc-code">b_user</span> с пагинацией.
    </p>

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
    </div>

    <?php if (!empty($pagination)): ?>
        <div style="margin-top: 20px; display: flex; gap: 8px; align-items: center; flex-wrap: wrap;">
            <?php if (!empty($pagination['has_prev'])): ?>
                <a
                    href="<?= htmlspecialcharsbx($pageUrl . '?page=' . (int)$pagination['prev_page']) ?>"
                    style="padding: 8px 12px; border: 1px solid #d1d5db; border-radius: 8px; text-decoration: none;"
                >
                    ← Назад
                </a>
            <?php endif; ?>

            <?php foreach (($pagination['pages'] ?? []) as $page): ?>
                <?php $isCurrent = ((int)$page === (int)($pagination['page'] ?? 1)); ?>

                <a
                    href="<?= htmlspecialcharsbx($pageUrl . '?page=' . (int)$page) ?>"
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
                    href="<?= htmlspecialcharsbx($pageUrl . '?page=' . (int)$pagination['next_page']) ?>"
                    style="padding: 8px 12px; border: 1px solid #d1d5db; border-radius: 8px; text-decoration: none;"
                >
                    Вперёд →
                </a>
            <?php endif; ?>
        </div>
    <?php endif; ?>
</div>


---

6. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/admin/users

Потом:

https://bitrix24-stage.gaz.ru/local/mvc_demo/admin/users?page=2

Если пользователей много, увидишь разные страницы.


---

Что мы сделали

Теперь список пользователей не грузит всё сразу.

Путь стал такой:

Controller
  ↓
берёт page из Request
  ↓
UserService
  ↓
Paginator считает limit и offset
  ↓
User Model получает нужный кусок пользователей
  ↓
View показывает таблицу и кнопки страниц

Главная мысль:

Paginator не знает про пользователей.
Он просто умеет считать:
- какая страница
- сколько всего записей
- сколько показывать
- какой offset

То есть потом его можно использовать где угодно:

список сайтов
список страниц
список заявок
список файлов
список отзывов

Дальше логично сделать поиск и фильтрацию для этой таблицы пользователей.