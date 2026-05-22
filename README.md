Идём дальше. Сейчас делаем админский раздел через prefix-группу.

То есть хотим такие адреса:

/local/mvc_demo/admin/dashboard
/local/mvc_demo/admin/users

И чтобы весь раздел /admin/... был доступен только администратору.


---

Что мы сейчас сделаем

Добавим:

/local/mvc_demo/Controllers/AdminController.php
/local/mvc_demo/Views/admin/dashboard.php
/local/mvc_demo/Views/admin/users.php

И обновим:

/local/mvc_demo/routes.php
/local/mvc_demo/Views/layouts/app.php


---

1. Создай /local/mvc_demo/Controllers/AdminController.php

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\Auth;
use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Response;

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
        ]);
    }

    public function users(): Response
    {
        return $this->render('admin/users', [
            'title' => 'Пользователи',
            'users' => [
                [
                    'id' => Auth::id(),
                    'login' => Auth::login(),
                    'name' => Auth::name(),
                    'email' => Auth::email(),
                    'is_admin' => Auth::isAdmin() ? 'Да' : 'Нет',
                ],
            ],
        ]);
    }
}

Пока список пользователей тестовый — выводим текущего пользователя. Позже подключим нормальную модель и будем брать пользователей из базы Битрикса.


---

2. Создай папку Views для админки

/local/mvc_demo/Views/admin/


---

3. Создай /local/mvc_demo/Views/admin/dashboard.php

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
        </ol>
    </div>
</div>


---

4. Создай /local/mvc_demo/Views/admin/users.php

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
        Пока это тестовый список. Позже здесь будет нормальная таблица пользователей.
    </p>

    <div class="mvc-info">
        <table style="width: 100%; border-collapse: collapse;">
            <thead>
                <tr>
                    <th style="text-align: left; padding: 8px; border-bottom: 1px solid #e5e7eb;">ID</th>
                    <th style="text-align: left; padding: 8px; border-bottom: 1px solid #e5e7eb;">Логин</th>
                    <th style="text-align: left; padding: 8px; border-bottom: 1px solid #e5e7eb;">Имя</th>
                    <th style="text-align: left; padding: 8px; border-bottom: 1px solid #e5e7eb;">Email</th>
                    <th style="text-align: left; padding: 8px; border-bottom: 1px solid #e5e7eb;">Админ</th>
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
                            <?= htmlspecialcharsbx($user['name'] ?? '') ?>
                        </td>

                        <td style="padding: 8px; border-bottom: 1px solid #e5e7eb;">
                            <?= htmlspecialcharsbx($user['email'] ?? '') ?>
                        </td>

                        <td style="padding: 8px; border-bottom: 1px solid #e5e7eb;">
                            <?= htmlspecialcharsbx($user['is_admin'] ?? '') ?>
                        </td>
                    </tr>
                <?php endforeach; ?>
            </tbody>
        </table>
    </div>
</div>


---

5. Обнови /local/mvc_demo/routes.php

Полностью замени файл:

<?php

use Local\Mvc\Core\Router;
use Local\MvcDemo\Controllers\HomeController;
use Local\MvcDemo\Controllers\AdminController;

/** @var Router $router */

/**
 * Публичные маршруты.
 */
$router->get('/', [HomeController::class, 'index']);

$router->get('/about', [HomeController::class, 'about']);

$router->get('/ping', [HomeController::class, 'ping']);

/**
 * Только авторизованные пользователи.
 */
$router->group(['middleware' => ['auth']], function (Router $router) {
    $router->get('/me', [HomeController::class, 'me']);

    /**
     * Админский раздел.
     *
     * Всё внутри получит:
     * prefix: /admin
     * middleware: auth + admin
     *
     * То есть:
     * /admin/dashboard
     * /admin/users
     */
    $router->group([
        'prefix' => '/admin',
        'middleware' => ['admin'],
    ], function (Router $router) {
        $router->get('/dashboard', [AdminController::class, 'dashboard']);
        $router->get('/users', [AdminController::class, 'users']);
    });
});

Что здесь важно

Вот эта группа:

$router->group([
    'prefix' => '/admin',
    'middleware' => ['admin'],
], function (Router $router) {

означает:

Все маршруты внутри начинаются с /admin
и доступны только администратору.

То есть:

$router->get('/dashboard', ...)

превращается в:

/admin/dashboard

А:

$router->get('/users', ...)

превращается в:

/admin/users


---

6. Обнови меню /local/mvc_demo/Views/layouts/app.php

Замени файл:

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

?>

<div class="mvc-app">
    <div class="mvc-topbar">
        <div class="mvc-brand">
            <div class="mvc-brand__title">MVC Demo</div>
            <div class="mvc-brand__subtitle">Отдельный проект на общем MVC-фреймворке</div>
        </div>

        <nav class="mvc-nav">
            <a href="/local/mvc_demo/">Главная</a>
            <a href="/local/mvc_demo/about">О проекте</a>
            <a href="/local/mvc_demo/me">Я</a>
            <a href="/local/mvc_demo/admin/dashboard">Админка</a>
            <a href="/local/mvc_demo/admin/users">Пользователи</a>
            <a href="/local/mvc_demo/ping" target="_blank">Ping JSON</a>
        </nav>
    </div>

    <?= $content ?? '' ?>
</div>


---

7. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Проверь публичную страницу:

https://bitrix24-stage.gaz.ru/local/mvc_demo/

Проверь страницу текущего пользователя:

https://bitrix24-stage.gaz.ru/local/mvc_demo/me

Проверь админскую страницу:

https://bitrix24-stage.gaz.ru/local/mvc_demo/admin/dashboard

Проверь админский список пользователей:

https://bitrix24-stage.gaz.ru/local/mvc_demo/admin/users


---

Что мы сделали

Теперь у нас есть полноценная структура:

/local/mvc
  общий фреймворк

/local/mvc_demo
  отдельный проект

/local/mvc_demo/Controllers/HomeController.php
  публичные и пользовательские страницы

/local/mvc_demo/Controllers/AdminController.php
  админские страницы

И маршруты стали понятные:

/                 публичная главная
/about            публичная страница
/ping             публичный JSON
/me               только авторизованный
/admin/dashboard  только админ
/admin/users      только админ

Главная мысль:

prefix добавляет начало адреса
middleware защищает маршрут

То есть группа:

$router->group([
    'prefix' => '/admin',
    'middleware' => ['admin'],
], function (Router $router) {
    ...
});

это как сказать:

Всё внутри — админский раздел.

Следующий шаг — сделать нормальный View-helper, чтобы в шаблонах не писать руками htmlspecialcharsbx() и чтобы были функции e(), url(), asset().