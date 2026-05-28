Отлично, идём дальше.

Сейчас сделаем Service-слой.

Зачем нужен Service

Смотри, сейчас у нас цепочка такая:

Controller → Model → Db

Это уже хорошо.

Но есть проблема: если в контроллере начнёт появляться много логики, он снова станет большим.

Например:

$users = User::latest(10);

foreach ($users as &$user) {
    $user['FULL_NAME'] = User::fullName($user);
    $user['ACTIVE_LABEL'] = $user['ACTIVE'] === 'Y' ? 'Да' : 'Нет';
}

Это уже не совсем задача контроллера.

Простыми словами:

Model — просто достаёт данные из таблицы.
Service — готовит данные для задачи.
Controller — командует процессом.
View — показывает результат.


---

1. Создай папку Services

/local/mvc_demo/Services/


---

2. Создай /local/mvc_demo/Services/UserService.php

<?php

namespace Local\MvcDemo\Services;

use Local\MvcDemo\Models\User;

/**
 * UserService
 *
 * Сервис для работы с пользователями.
 *
 * Model просто достаёт данные.
 * Service подготавливает эти данные для контроллера и view.
 */
class UserService
{
    /**
     * Данные для админского dashboard.
     */
    public function dashboardStats(): array
    {
        return [
            'users_count' => User::count(),
        ];
    }

    /**
     * Последние пользователи для таблицы.
     */
    public function latestForTable(int $limit = 10): array
    {
        $users = User::latest($limit);

        $result = [];

        foreach ($users as $user) {
            $result[] = $this->prepareForTable($user);
        }

        return $result;
    }

    /**
     * Найти пользователя для карточки.
     */
    public function findForDetail(int $id): ?array
    {
        $user = User::findForAdmin($id);

        if (!$user) {
            return null;
        }

        return $this->prepareForDetail($user);
    }

    /**
     * Подготовить пользователя для таблицы.
     */
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

    /**
     * Подготовить пользователя для карточки.
     */
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

3. Обнови /local/mvc_demo/Controllers/AdminController.php

Полностью замени файл:

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\Auth;
use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Response;
use Local\MvcDemo\Services\UserService;

class AdminController extends Controller
{
    public function dashboard(): Response
    {
        $userService = new UserService();

        return $this->render('admin/dashboard', [
            'title' => 'Админ-панель',
            'message' => 'Это защищённая админская страница. Сюда может зайти только администратор.',
            'user' => [
                'id' => Auth::id(),
                'login' => Auth::login(),
                'name' => Auth::name(),
                'email' => Auth::email(),
            ],
            'stats' => $userService->dashboardStats(),
        ]);
    }

    public function users(): Response
    {
        $userService = new UserService();

        return $this->render('admin/users', [
            'title' => 'Пользователи',
            'users' => $userService->latestForTable(10),
        ]);
    }

    public function userDetail(string $id): Response
    {
        $userService = new UserService();

        $user = $userService->findForDetail((int)$id);

        if (!$user) {
            return Response::html(
                '<h1>404</h1><p>Пользователь не найден.</p>',
                404
            );
        }

        return $this->render('admin/user_detail', [
            'title' => 'Карточка пользователя',
            'user' => $user,
        ]);
    }
}


---

4. Обнови /local/mvc_demo/Views/admin/dashboard.php

Найди блок, где выводится количество пользователей.
Если у тебя было:

<?= htmlspecialcharsbx($usersCount ?? 0) ?>

замени на:

<?= htmlspecialcharsbx($stats['users_count'] ?? 0) ?>

Полный файл может быть таким:

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

?>

<div class="mvc-card">
    <h1 class="mvc-page-title">
        <?= htmlspecialcharsbx($title ?? 'Админ-панель') ?>
    </h1>

    <p class="mvc-page-text">
        <?= htmlspecialcharsbx($message ?? '') ?>
    </p>

    <div class="mvc-info">
        <b>Текущий администратор:</b>

        <ol>
            <li>
                ID:
                <span class="mvc-code">
                    <?= htmlspecialcharsbx($user['id'] ?? '') ?>
                </span>
            </li>

            <li>
                Логин:
                <span class="mvc-code">
                    <?= htmlspecialcharsbx($user['login'] ?? '') ?>
                </span>
            </li>

            <li>
                Имя:
                <span class="mvc-code">
                    <?= htmlspecialcharsbx($user['name'] ?? '') ?>
                </span>
            </li>

            <li>
                Email:
                <span class="mvc-code">
                    <?= htmlspecialcharsbx($user['email'] ?? '') ?>
                </span>
            </li>

            <li>
                Всего пользователей:
                <span class="mvc-code">
                    <?= htmlspecialcharsbx($stats['users_count'] ?? 0) ?>
                </span>
            </li>
        </ol>
    </div>
</div>


---

5. Обнови /local/mvc_demo/Views/admin/users.php

Теперь view больше не вызывает модель User::fullName().
Она просто показывает уже готовые данные.

Полностью замени файл:

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

?>

<div class="mvc-card">
    <h1 class="mvc-page-title">
        <?= htmlspecialcharsbx($title ?? 'Пользователи') ?>
    </h1>

    <p class="mvc-page-text">
        Это список последних пользователей из таблицы <span class="mvc-code">b_user</span>.
    </p>

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
</div>


---

6. Обнови /local/mvc_demo/Views/admin/user_detail.php

Полностью замени файл:

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

?>

<div class="mvc-card">
    <h1 class="mvc-page-title">
        <?= htmlspecialcharsbx($title ?? 'Карточка пользователя') ?>
    </h1>

    <p class="mvc-page-text">
        Данные пользователя получены через модель и подготовлены через сервис.
    </p>

    <div class="mvc-info">
        <b><?= htmlspecialcharsbx($user['full_name'] ?? '') ?></b>

        <ol>
            <li>
                ID:
                <span class="mvc-code"><?= htmlspecialcharsbx($user['id'] ?? '') ?></span>
            </li>

            <li>
                Логин:
                <span class="mvc-code"><?= htmlspecialcharsbx($user['login'] ?? '') ?></span>
            </li>

            <li>
                Email:
                <span class="mvc-code"><?= htmlspecialcharsbx($user['email'] ?? '') ?></span>
            </li>

            <li>
                Активен:
                <span class="mvc-code"><?= htmlspecialcharsbx($user['active_label'] ?? '') ?></span>
            </li>

            <li>
                Дата регистрации:
                <span class="mvc-code"><?= htmlspecialcharsbx($user['date_register'] ?? '') ?></span>
            </li>

            <li>
                Последний вход:
                <span class="mvc-code"><?= htmlspecialcharsbx($user['last_login'] ?? '') ?></span>
            </li>
        </ol>
    </div>

    <div style="margin-top: 20px;">
        <a href="/local/mvc_demo/admin/users">
            ← Назад к списку
        </a>
    </div>
</div>


---

7. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Проверь:

/local/mvc_demo/admin/dashboard
/local/mvc_demo/admin/users
/local/mvc_demo/admin/users/1


---

Что мы сделали

Теперь цепочка стала ещё правильнее:

Route
  ↓
Controller
  ↓
Service
  ↓
Model
  ↓
Db
  ↓
View

Простыми словами:

Model — знает таблицу.
Service — готовит данные для задачи.
Controller — вызывает нужный сервис.
View — просто показывает.

Почему это важно

Для маленькой страницы можно обойтись без Service.

Но в реальных проектах он очень нужен.

Например для sitebuilder:

Site model       — работает с таблицей sites
Page model       — работает с таблицей pages
Block model      — работает с таблицей blocks

SiteService      — создать сайт, выдать права, создать главную страницу
PageService      — создать страницу, проверить slug, сохранить layout
DiskService      — создать папку в Диске, синхронизировать права

То есть бизнес-логика будет не в контроллере, а в сервисах.