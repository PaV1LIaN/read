Да, тогда не делаем View-helper.
Идём дальше проще: учимся обрабатывать формы и POST-запросы.

Сейчас сделаем страницу:

/local/mvc_demo/form

На ней будет форма:

Имя
Сообщение
Кнопка отправить

Форма будет отправляться сюда:

POST /local/mvc_demo/form/send

И контроллер проверит:

имя заполнено?
сообщение заполнено?
sessid Битрикса правильный?


---

1. Создай контроллер формы

Файл:

/local/mvc_demo/Controllers/FormController.php

Код:

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Response;

class FormController extends Controller
{
    /**
     * Показать форму.
     */
    public function index(): Response
    {
        return $this->render('form/index', [
            'title' => 'Тестовая форма',
            'errors' => [],
            'success' => '',
            'old' => [
                'name' => '',
                'message' => '',
            ],
        ]);
    }

    /**
     * Обработать отправку формы.
     */
    public function send(): Response
    {
        $name = trim((string)$this->request->post('name', ''));
        $message = trim((string)$this->request->post('message', ''));

        $errors = [];

        /**
         * Проверяем sessid Битрикса.
         *
         * Это защита от чужой отправки формы.
         */
        if (function_exists('check_bitrix_sessid') && !check_bitrix_sessid()) {
            $errors[] = 'Ошибка безопасности: неверный sessid.';
        }

        if ($name === '') {
            $errors[] = 'Введите имя.';
        }

        if ($message === '') {
            $errors[] = 'Введите сообщение.';
        }

        if (!empty($errors)) {
            return $this->render('form/index', [
                'title' => 'Тестовая форма',
                'errors' => $errors,
                'success' => '',
                'old' => [
                    'name' => $name,
                    'message' => $message,
                ],
            ]);
        }

        /**
         * Пока никуда не сохраняем.
         * Просто показываем, что POST-запрос успешно обработан.
         */
        return $this->render('form/index', [
            'title' => 'Тестовая форма',
            'errors' => [],
            'success' => 'Форма успешно отправлена. Имя: ' . $name . ', сообщение: ' . $message,
            'old' => [
                'name' => '',
                'message' => '',
            ],
        ]);
    }
}


---

2. Создай папку view

/local/mvc_demo/Views/form/


---

3. Создай view формы

Файл:

/local/mvc_demo/Views/form/index.php

Код:

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

$name = $old['name'] ?? '';
$message = $old['message'] ?? '';

?>

<div class="mvc-card">
    <h1 class="mvc-page-title">
        <?= htmlspecialcharsbx($title ?? 'Форма') ?>
    </h1>

    <p class="mvc-page-text">
        Это простая форма, чтобы проверить POST-запросы в нашем MVC.
    </p>

    <?php if (!empty($errors)): ?>
        <div class="mvc-info" style="border-color: #fecaca; background: #fef2f2;">
            <b style="color: #991b1b;">Ошибки:</b>

            <ol>
                <?php foreach ($errors as $error): ?>
                    <li style="color: #991b1b;">
                        <?= htmlspecialcharsbx($error) ?>
                    </li>
                <?php endforeach; ?>
            </ol>
        </div>
    <?php endif; ?>

    <?php if (!empty($success)): ?>
        <div class="mvc-info" style="border-color: #bbf7d0; background: #f0fdf4;">
            <b style="color: #166534;">Успешно:</b>

            <p style="color: #166534; margin-bottom: 0;">
                <?= htmlspecialcharsbx($success) ?>
            </p>
        </div>
    <?php endif; ?>

    <form method="post" action="/local/mvc_demo/form/send" style="margin-top: 24px;">
        <?php if (function_exists('bitrix_sessid_post')): ?>
            <?= bitrix_sessid_post() ?>
        <?php endif; ?>

        <div style="margin-bottom: 16px;">
            <label style="display: block; margin-bottom: 6px; font-weight: 600;">
                Имя
            </label>

            <input
                type="text"
                name="name"
                value="<?= htmlspecialcharsbx($name) ?>"
                style="width: 100%; min-height: 42px; padding: 8px 12px; border: 1px solid #d1d5db; border-radius: 10px;"
            >
        </div>

        <div style="margin-bottom: 16px;">
            <label style="display: block; margin-bottom: 6px; font-weight: 600;">
                Сообщение
            </label>

            <textarea
                name="message"
                rows="5"
                style="width: 100%; padding: 8px 12px; border: 1px solid #d1d5db; border-radius: 10px;"
            ><?= htmlspecialcharsbx($message) ?></textarea>
        </div>

        <button
            type="submit"
            style="min-height: 42px; padding: 0 18px; border: 0; border-radius: 10px; background: #2563eb; color: #fff; font-weight: 600; cursor: pointer;"
        >
            Отправить
        </button>
    </form>
</div>


---

4. Обнови routes

Файл:

/local/mvc_demo/routes.php

Добавь use:

use Local\MvcDemo\Controllers\FormController;

И добавь маршруты:

$router->get('/form', [FormController::class, 'index']);

$router->post('/form/send', [FormController::class, 'send']);

Примерно получится так:

<?php

use Local\Mvc\Core\Router;
use Local\MvcDemo\Controllers\HomeController;
use Local\MvcDemo\Controllers\AdminController;
use Local\MvcDemo\Controllers\FormController;

/** @var Router $router */

/**
 * Публичные маршруты.
 */
$router->get('/', [HomeController::class, 'index']);

$router->get('/about', [HomeController::class, 'about']);

$router->get('/ping', [HomeController::class, 'ping']);

$router->get('/form', [FormController::class, 'index']);

$router->post('/form/send', [FormController::class, 'send']);

/**
 * Только авторизованные пользователи.
 */
$router->group(['middleware' => ['auth']], function (Router $router) {
    $router->get('/me', [HomeController::class, 'me']);

    $router->group([
        'prefix' => '/admin',
        'middleware' => ['admin'],
    ], function (Router $router) {
        $router->get('/dashboard', [AdminController::class, 'dashboard']);
        $router->get('/users', [AdminController::class, 'users']);
    });
});


---

5. Добавь ссылку в меню

Файл:

/local/mvc_demo/Views/layouts/app.php

В меню добавь:

<a href="/local/mvc_demo/form">Форма</a>

Например:

<nav class="mvc-nav">
    <a href="/local/mvc_demo/">Главная</a>
    <a href="/local/mvc_demo/about">О проекте</a>
    <a href="/local/mvc_demo/form">Форма</a>
    <a href="/local/mvc_demo/me">Я</a>
    <a href="/local/mvc_demo/admin/dashboard">Админка</a>
    <a href="/local/mvc_demo/admin/users">Пользователи</a>
    <a href="/local/mvc_demo/ping" target="_blank">Ping JSON</a>
</nav>


---

6. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/form

Сначала нажми “Отправить” с пустыми полями. Должны появиться ошибки:

Введите имя.
Введите сообщение.

Потом заполни поля и отправь. Должно появиться сообщение:

Форма успешно отправлена...


---

Что мы сейчас добавили

Раньше у нас были только GET-запросы:

открыл страницу
получил HTML или JSON

Теперь появился POST:

пользователь заполнил форму
нажал кнопку
браузер отправил POST
Router нашёл POST-маршрут
Controller проверил данные
View показал результат

Главное, что нужно понять:

GET  — показать страницу
POST — обработать действие

Например:

GET  /form       — показать форму
POST /form/send  — обработать форму

Следующим шагом можно сделать валидацию отдельным инструментом, чтобы проверки Введите имя, Введите сообщение не писать вручную в каждом контроллере.