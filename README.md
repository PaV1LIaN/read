Идём дальше. Табличный компонент пока не делаем, чтобы не путаться. Следующий важный шаг — API для AJAX.

В реальных проектах у нас будут страницы и API:

Страница:
GET /local/sitebuilder/sites

API:
GET  /local/sitebuilder/api/sites
POST /local/sitebuilder/api/sites/create
POST /local/sitebuilder/api/sites/delete

Сейчас сделаем API в mvc_demo:

/local/mvc_demo/api/users
/local/mvc_demo/api/users/1


---

1. Добавляем JSON в Request

Открой файл:

/local/mvc/Core/Request.php

Внутрь класса добавь свойства:

private ?string $rawBody = null;
private ?array $jsonBody = null;

Чтобы начало класса было примерно такое:

private array $get;
private array $post;
private array $server;
private array $files;
private array $routeParams = [];

private ?string $rawBody = null;
private ?array $jsonBody = null;

Потом перед последней } класса добавь методы:

/**
 * Сырой body запроса.
 *
 * Нужно для JSON-запросов.
 */
public function rawBody(): string
{
    if ($this->rawBody !== null) {
        return $this->rawBody;
    }

    $body = file_get_contents('php://input');

    $this->rawBody = is_string($body) ? $body : '';

    return $this->rawBody;
}

/**
 * JSON из body запроса.
 *
 * Например, если фронт отправил:
 * {"name":"Тест"}
 *
 * То:
 * $request->json('name')
 * вернёт "Тест".
 */
public function json(string $key, mixed $default = null): mixed
{
    $data = $this->jsonAll();

    return $data[$key] ?? $default;
}

/**
 * Весь JSON body как массив.
 */
public function jsonAll(): array
{
    if ($this->jsonBody !== null) {
        return $this->jsonBody;
    }

    $raw = trim($this->rawBody());

    if ($raw === '') {
        $this->jsonBody = [];
        return $this->jsonBody;
    }

    $decoded = json_decode($raw, true);

    $this->jsonBody = is_array($decoded) ? $decoded : [];

    return $this->jsonBody;
}

/**
 * Это AJAX-запрос?
 */
public function isAjax(): bool
{
    $requestedWith = (string)$this->server('HTTP_X_REQUESTED_WITH', '');

    return strtolower($requestedWith) === 'xmlhttprequest';
}


---

2. Создаём API-контроллер пользователей

Создай файл:

/local/mvc_demo/Controllers/UserApiController.php

Код:

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Response;
use Local\MvcDemo\Services\UserService;

class UserApiController extends Controller
{
    /**
     * GET /api/users
     *
     * Список пользователей JSON.
     */
    public function index(): Response
    {
        $page = (int)$this->request->get('page', 1);
        $search = trim((string)$this->request->get('q', ''));

        $userService = new UserService();

        $result = $userService->paginateForTable($page, 10, $search);

        return $this->success([
            'items' => $result['items'],
            'pagination' => $result['pagination'],
            'search' => $result['search'],
        ]);
    }

    /**
     * GET /api/users/{id}
     *
     * Один пользователь JSON.
     */
    public function show(string $id): Response
    {
        $userService = new UserService();

        $user = $userService->findForDetail((int)$id);

        if (!$user) {
            return $this->error('USER_NOT_FOUND', [
                'message' => 'Пользователь не найден',
                'id' => (int)$id,
            ], 404);
        }

        return $this->success([
            'user' => $user,
        ]);
    }
}


---

3. Обновляем routes

Открой:

/local/mvc_demo/routes.php

Добавь use:

use Local\MvcDemo\Controllers\UserApiController;

И ниже добавь API-группу:

/**
 * API.
 *
 * Пока API пользователей доступен только админам.
 */
$router->group([
    'prefix' => '/api',
    'middleware' => ['auth', 'admin'],
], function (Router $router) {
    $router->get('/users', [UserApiController::class, 'index']);
    $router->get('/users/{id:\d+}', [UserApiController::class, 'show']);
});

Полностью файл должен быть примерно такой:

<?php

use Local\Mvc\Core\Router;
use Local\MvcDemo\Controllers\HomeController;
use Local\MvcDemo\Controllers\AdminController;
use Local\MvcDemo\Controllers\FormController;
use Local\MvcDemo\Controllers\UserApiController;

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
        $router->get('/users/{id:\d+}', [AdminController::class, 'userDetail']);
    });
});

/**
 * API.
 */
$router->group([
    'prefix' => '/api',
    'middleware' => ['auth', 'admin'],
], function (Router $router) {
    $router->get('/users', [UserApiController::class, 'index']);
    $router->get('/users/{id:\d+}', [UserApiController::class, 'show']);
});


---

4. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/api/users

Должен быть JSON:

{
  "ok": true,
  "data": {
    "items": [],
    "pagination": {
      "total": 10,
      "page": 1
    },
    "search": ""
  }
}

Проверь поиск:

https://bitrix24-stage.gaz.ru/local/mvc_demo/api/users?q=admin

Проверь одного пользователя:

https://bitrix24-stage.gaz.ru/local/mvc_demo/api/users/1

Если пользователя нет, будет:

{
  "ok": false,
  "error": "USER_NOT_FOUND",
  "details": {
    "message": "Пользователь не найден",
    "id": 1
  }
}


---

Что мы сделали

Теперь у проекта есть два типа маршрутов:

HTML-страницы:
GET /admin/users

JSON API:
GET /api/users
GET /api/users/{id}

Путь такой:

Браузер / AJAX
  ↓
Router
  ↓
Middleware auth/admin
  ↓
UserApiController
  ↓
UserService
  ↓
User Model
  ↓
JSON Response

Главная мысль:

Обычный Controller может возвращать HTML.
API Controller возвращает JSON.
Service и Model при этом используются те же самые.

Это правильно: одну бизнес-логику можно использовать и для страницы, и для API.