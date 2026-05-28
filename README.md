Идём дальше. Сейчас сделаем CSRF-защиту как middleware.

Простыми словами:

CSRF — это проверка, что форму отправил именно наш сайт, а не чужая страница.

В Битриксе за это отвечает sessid.

Сейчас мы проверяем sessid прямо в FormController:

if (function_exists('check_bitrix_sessid') && !check_bitrix_sessid()) {
    $errors[] = 'Ошибка безопасности: неверный sessid.';
}

Но это неправильно для фреймворка. Лучше так:

$router->post('/form/send', [FormController::class, 'send'], ['csrf']);

То есть маршрут сам говорит:

Перед обработкой формы проверь sessid.


---

1. Замени /local/mvc/Core/Middleware.php

<?php

namespace Local\Mvc\Core;

/**
 * Middleware
 *
 * Проверки, которые выполняются ДО контроллера.
 */
class Middleware
{
    public static function handle(array $middlewares, Request $request): ?Response
    {
        foreach ($middlewares as $middleware) {
            $middleware = trim((string)$middleware);

            if ($middleware === '') {
                continue;
            }

            $response = self::handleOne($middleware, $request);

            if ($response instanceof Response) {
                return $response;
            }
        }

        return null;
    }

    private static function handleOne(string $middleware, Request $request): ?Response
    {
        /**
         * auth — только авторизованные пользователи.
         */
        if ($middleware === 'auth') {
            if (Auth::check()) {
                return null;
            }

            return Response::json([
                'ok' => false,
                'error' => 'AUTH_REQUIRED',
                'details' => [
                    'message' => 'Нужно авторизоваться',
                ],
            ], 401);
        }

        /**
         * admin — только администраторы.
         */
        if ($middleware === 'admin') {
            if (Auth::isAdmin()) {
                return null;
            }

            return Response::json([
                'ok' => false,
                'error' => 'ADMIN_REQUIRED',
                'details' => [
                    'message' => 'Нужны права администратора',
                ],
            ], 403);
        }

        /**
         * csrf — проверка sessid Битрикса.
         *
         * Используем для POST-запросов:
         * создание, сохранение, удаление.
         */
        if ($middleware === 'csrf') {
            if ($request->method() !== 'POST') {
                return null;
            }

            if (function_exists('check_bitrix_sessid') && check_bitrix_sessid()) {
                return null;
            }

            return Response::html(
                '<h1>403</h1>'
                . '<p>Ошибка безопасности.</p>'
                . '<p>Неверный sessid. Обновите страницу и попробуйте ещё раз.</p>',
                403
            );
        }

        /**
         * Неизвестный middleware — ошибка разработчика.
         */
        return Response::json([
            'ok' => false,
            'error' => 'UNKNOWN_MIDDLEWARE',
            'details' => [
                'middleware' => $middleware,
            ],
        ], 500);
    }
}


---

2. Обнови /local/mvc_demo/routes.php

Найди маршрут:

$router->post('/form/send', [FormController::class, 'send']);

Замени на:

$router->post('/form/send', [FormController::class, 'send'], ['csrf']);

Полный пример:

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

$router->post('/form/send', [FormController::class, 'send'], ['csrf']);

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

3. Убери проверку sessid из FormController

Файл:

/local/mvc_demo/Controllers/FormController.php

Замени полностью:

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Flash;
use Local\Mvc\Core\Response;
use Local\Mvc\Core\Validator;

class FormController extends Controller
{
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

    public function send(): Response
    {
        $data = [
            'name' => trim((string)$this->request->post('name', '')),
            'message' => trim((string)$this->request->post('message', '')),
        ];

        $errors = [];

        $validator = Validator::make($data)
            ->required('name', 'Введите имя.')
            ->min('name', 2, 'Имя должно быть не короче 2 символов.')
            ->max('name', 100, 'Имя должно быть не длиннее 100 символов.')
            ->required('message', 'Введите сообщение.')
            ->min('message', 5, 'Сообщение должно быть не короче 5 символов.')
            ->max('message', 1000, 'Сообщение должно быть не длиннее 1000 символов.');

        if ($validator->fails()) {
            $errors = array_merge($errors, $validator->errorList());
        }

        if (!empty($errors)) {
            Flash::old($data);

            foreach ($errors as $error) {
                Flash::error($error);
            }

            return $this->redirectBack('/form');
        }

        Flash::success('Форма успешно отправлена. Имя: ' . $data['name']);

        return $this->redirectBack('/form');
    }
}


---

4. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/form

Отправь форму.

Должно работать так же, как раньше:

ошибка в полях → красные сообщения + значения остались
успешно → зелёное сообщение + поля очистились


---

Что мы сделали

Раньше контроллер сам проверял безопасность:

check_bitrix_sessid()

Теперь безопасность формы проверяет маршрут:

$router->post('/form/send', [FormController::class, 'send'], ['csrf']);

То есть теперь схема такая:

POST /form/send
  ↓
Router
  ↓
Middleware csrf проверяет sessid
  ↓
если всё хорошо — FormController::send()
  ↓
Validator проверяет поля
  ↓
Flash + redirect

Главная мысль:

Controller занимается логикой формы.
Middleware занимается проверкой доступа и безопасности.

Это правильно для фреймворка.