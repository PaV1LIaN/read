Идём дальше. Сейчас сделаем Model нормально.

Простыми словами:

Controller не должен сам писать SQL.
Model должна отвечать за данные.

Сейчас в AdminController у нас тестовый список пользователей. Сделаем правильно:

AdminController → User model → база Битрикса → View


---

1. Замени /local/mvc/Core/Model.php

<?php

namespace Local\Mvc\Core;

use RuntimeException;

/**
 * Model
 *
 * Базовая модель.
 *
 * Простыми словами:
 * это родитель для всех моделей проекта.
 */
abstract class Model
{
    /**
     * Имя таблицы.
     *
     * Например:
     * protected static string $table = 'b_user';
     */
    protected static string $table = '';

    /**
     * Главный ключ.
     */
    protected static string $primaryKey = 'ID';

    protected static function table(): string
    {
        if (static::$table === '') {
            throw new RuntimeException('У модели не указана таблица: ' . static::class);
        }

        return static::$table;
    }

    public static function all(int $limit = 100): array
    {
        $limit = max(1, min($limit, 500));

        return Db::fetchAll(
            'SELECT * FROM ' . static::table() . '
             ORDER BY ' . static::$primaryKey . ' DESC
             LIMIT ' . $limit
        );
    }

    public static function find(int|string $id): ?array
    {
        return Db::fetchOne(
            'SELECT * FROM ' . static::table() . '
             WHERE ' . static::$primaryKey . ' = :id
             LIMIT 1',
            [
                'id' => $id,
            ]
        );
    }

    public static function count(): int
    {
        return (int)Db::value(
            'SELECT COUNT(*) FROM ' . static::table()
        );
    }

    public static function deleteById(int|string $id): bool
    {
        return Db::execute(
            'DELETE FROM ' . static::table() . '
             WHERE ' . static::$primaryKey . ' = :id',
            [
                'id' => $id,
            ]
        );
    }
}


---

2. Создай папку моделей проекта

/local/mvc_demo/Models/


---

3. Создай /local/mvc_demo/Models/User.php

<?php

namespace Local\MvcDemo\Models;

use Local\Mvc\Core\Db;
use Local\Mvc\Core\Model;

/**
 * User
 *
 * Модель пользователя Битрикса.
 *
 * Работает с таблицей b_user.
 */
class User extends Model
{
    protected static string $table = 'b_user';

    protected static string $primaryKey = 'ID';

    public static function latest(int $limit = 10): array
    {
        $limit = max(1, min($limit, 100));

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
            LIMIT {$limit}
        ");
    }

    public static function findForAdmin(int $id): ?array
    {
        return Db::fetchOne("
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
            WHERE ID = :id
            LIMIT 1
        ", [
            'id' => $id,
        ]);
    }

    public static function fullName(array $user): string
    {
        $lastName = trim((string)($user['LAST_NAME'] ?? ''));
        $name = trim((string)($user['NAME'] ?? ''));
        $secondName = trim((string)($user['SECOND_NAME'] ?? ''));

        $fullName = trim($lastName . ' ' . $name . ' ' . $secondName);

        if ($fullName !== '') {
            return $fullName;
        }

        return (string)($user['LOGIN'] ?? '');
    }
}


---

4. Обнови /local/mvc_demo/Controllers/AdminController.php

Полностью замени файл:

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\Auth;
use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Response;
use Local\MvcDemo\Models\User;

class AdminController extends Controller
{
    public function dashboard(): Response
    {
        return $this->render('admin/dashboard', [
            'title' => 'Админ-панель',
            'message' => 'Это защищённая админская страница. Сюда может зайти только администратор.',
            'user' => [
                'id' => Auth::id(),
                'login' => Auth::login(),
                'name' => Auth::name(),
                'email' => Auth::email(),
            ],
            'usersCount' => User::count(),
        ]);
    }

    public function users(): Response
    {
        $users = User::latest(10);

        return $this->render('admin/users', [
            'title' => 'Пользователи',
            'users' => $users,
        ]);
    }

    public function userDetail(string $id): Response
    {
        $user = User::findForAdmin((int)$id);

        if (!$user) {
            return Response::html(
                '<h1>404</h1><p>Пользователь не найден.</p>',
                404
            );
        }

        return $this->render('admin/user_detail', [
            'title' => 'Карточка пользователя',
            'user' => $user,
            'fullName' => User::fullName($user),
        ]);
    }
}


---

5. Обнови /local/mvc_demo/Views/admin/dashboard.php

Внутри блока <ol> после email добавь:

<li>
    Всего пользователей:
    <span class="mvc-code">
        <?= htmlspecialcharsbx($usersCount ?? 0) ?>
    </span>
</li>

То есть теперь dashboard будет показывать количество пользователей через модель.


---

6. Замени /local/mvc_demo/Views/admin/users.php

<?php

use Local\MvcDemo\Models\User;

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
                            <?= htmlspecialcharsbx($user['ID'] ?? '') ?>
                        </td>

                        <td style="padding: 8px; border-bottom: 1px solid #e5e7eb;">
                            <?= htmlspecialcharsbx($user['LOGIN'] ?? '') ?>
                        </td>

                        <td style="padding: 8px; border-bottom: 1px solid #e5e7eb;">
                            <?= htmlspecialcharsbx(User::fullName($user)) ?>
                        </td>

                        <td style="padding: 8px; border-bottom: 1px solid #e5e7eb;">
                            <?= htmlspecialcharsbx($user['EMAIL'] ?? '') ?>
                        </td>

                        <td style="padding: 8px; border-bottom: 1px solid #e5e7eb;">
                            <?= (($user['ACTIVE'] ?? '') === 'Y') ? 'Да' : 'Нет' ?>
                        </td>

                        <td style="padding: 8px; border-bottom: 1px solid #e5e7eb;">
                            <a href="/local/mvc_demo/admin/users/<?= (int)($user['ID'] ?? 0) ?>">
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

7. Замени /local/mvc_demo/Views/admin/user_detail.php

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
        Данные пользователя получены через модель <span class="mvc-code">User</span>.
    </p>

    <div class="mvc-info">
        <b><?= htmlspecialcharsbx($fullName ?? '') ?></b>

        <ol>
            <li>
                ID:
                <span class="mvc-code"><?= htmlspecialcharsbx($user['ID'] ?? '') ?></span>
            </li>

            <li>
                Логин:
                <span class="mvc-code"><?= htmlspecialcharsbx($user['LOGIN'] ?? '') ?></span>
            </li>

            <li>
                Email:
                <span class="mvc-code"><?= htmlspecialcharsbx($user['EMAIL'] ?? '') ?></span>
            </li>

            <li>
                Активен:
                <span class="mvc-code"><?= (($user['ACTIVE'] ?? '') === 'Y') ? 'Да' : 'Нет' ?></span>
            </li>

            <li>
                Дата регистрации:
                <span class="mvc-code"><?= htmlspecialcharsbx($user['DATE_REGISTER'] ?? '') ?></span>
            </li>

            <li>
                Последний вход:
                <span class="mvc-code"><?= htmlspecialcharsbx($user['LAST_LOGIN'] ?? '') ?></span>
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

8. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/admin/dashboard

Потом:

https://bitrix24-stage.gaz.ru/local/mvc_demo/admin/users

Потом нажми “Открыть” у пользователя.


---

Что мы сделали

Теперь у нас появилась настоящая MVC-цепочка:

Route
  ↓
AdminController
  ↓
User Model
  ↓
Db
  ↓
b_user
  ↓
View

Простыми словами:

Controller больше не знает, как устроена таблица.
Model отвечает за данные.
View только показывает.

Это уже прям настоящая основа для будущих проектов:

Site model
Page model
Block model
Application model
Report model

Например в sitebuilder потом будет так:

$sites = Site::latest();
$page = Page::find($id);
$blocks = Block::byPage($pageId);

Следующим шагом сделаем Repository/Service слой, чтобы понять разницу: где должна быть бизнес-логика, а где просто работа с таблицей.