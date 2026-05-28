Идём дальше. Сейчас сделаем базовый ApiController.

Зачем он нужен

Сейчас API-контроллеры наследуются от обычного:

Controller

И возвращают JSON через:

$this->success(...)
$this->error(...)

Но API у нас будет много:

/api/users
/api/sites
/api/pages
/api/files
/api/settings

Поэтому удобно сделать отдельного родителя:

ApiController

Он будет специально для JSON-ответов.


---

1. Создай /local/mvc/Core/ApiController.php

<?php

namespace Local\Mvc\Core;

/**
 * ApiController
 *
 * Базовый контроллер для API.
 *
 * Обычный Controller умеет HTML + JSON.
 * ApiController специально заточен под JSON.
 */
class ApiController extends Controller
{
    /**
     * Успешный JSON-ответ.
     */
    protected function ok(array $data = [], int $status = 200): Response
    {
        return $this->json([
            'ok' => true,
            'data' => $data,
        ], $status);
    }

    /**
     * JSON-ошибка.
     */
    protected function fail(string $error, array $details = [], int $status = 400): Response
    {
        return $this->json([
            'ok' => false,
            'error' => $error,
            'details' => $details,
        ], $status);
    }

    /**
     * Ошибка валидации.
     */
    protected function validationError(array $errors): Response
    {
        return $this->fail('VALIDATION_ERROR', [
            'errors' => $errors,
        ], 422);
    }

    /**
     * Не найдено.
     */
    protected function notFound(string $message = 'Запись не найдена', array $details = []): Response
    {
        return $this->fail('NOT_FOUND', array_merge([
            'message' => $message,
        ], $details), 404);
    }

    /**
     * Доступ запрещён.
     */
    protected function forbidden(string $message = 'Доступ запрещён'): Response
    {
        return $this->fail('FORBIDDEN', [
            'message' => $message,
        ], 403);
    }

    /**
     * Данные JSON-запроса.
     */
    protected function jsonData(): array
    {
        return $this->request->jsonAll();
    }
}


---

2. Обнови /local/mvc_demo/Controllers/UserApiController.php

Полностью замени файл:

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\ApiController;
use Local\Mvc\Core\Response;
use Local\MvcDemo\Services\UserService;

class UserApiController extends ApiController
{
    /**
     * GET /api/users
     */
    public function index(): Response
    {
        $page = (int)$this->request->get('page', 1);
        $search = trim((string)$this->request->get('q', ''));

        $userService = new UserService();

        $result = $userService->paginateForTable($page, 10, $search);

        return $this->ok([
            'items' => $result['items'],
            'pagination' => $result['pagination'],
            'search' => $result['search'],
        ]);
    }

    /**
     * GET /api/users/{id}
     */
    public function show(string $id): Response
    {
        $userService = new UserService();

        $user = $userService->findForDetail((int)$id);

        if (!$user) {
            return $this->notFound('Пользователь не найден', [
                'id' => (int)$id,
            ]);
        }

        return $this->ok([
            'user' => $user,
        ]);
    }
}


---

3. Обнови /local/mvc_demo/Controllers/AjaxDemoController.php

Я предлагаю переименовать метод echo() в echoText(), чтобы не путаться с языковой конструкцией echo.

Полностью замени файл:

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\ApiController;
use Local\Mvc\Core\Auth;
use Local\Mvc\Core\Response;

class AjaxDemoController extends ApiController
{
    /**
     * HTML-страница с AJAX-примером.
     */
    public function index(): Response
    {
        return $this->render('ajax/index', [
            'title' => 'AJAX Demo',
            'sessid' => function_exists('bitrix_sessid') ? bitrix_sessid() : '',
            'userId' => Auth::id(),
        ]);
    }

    /**
     * POST /api/ajax-demo/echo
     *
     * API принимает JSON и возвращает JSON.
     */
    public function echoText(): Response
    {
        $data = $this->jsonData();

        $text = trim((string)($data['text'] ?? ''));

        if ($text === '') {
            return $this->validationError([
                'text' => [
                    'Введите текст.',
                ],
            ]);
        }

        return $this->ok([
            'received_text' => $text,
            'length' => mb_strlen($text),
            'user_id' => Auth::id(),
            'time' => date('Y-m-d H:i:s'),
        ]);
    }
}


---

4. Обнови маршрут AJAX в /local/mvc_demo/routes.php

Найди:

$router->post('/ajax-demo/echo', [AjaxDemoController::class, 'echo'], ['csrf']);

Замени на:

$router->post('/ajax-demo/echo', [AjaxDemoController::class, 'echoText'], ['csrf']);


---

5. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Проверь API пользователей:

/local/mvc_demo/api/users

Проверь одного пользователя:

/local/mvc_demo/api/users/1

Проверь AJAX-страницу:

/local/mvc_demo/ajax-demo

Нажми кнопку:

Отправить AJAX

Должен вернуться JSON.


---

Что мы сделали

Раньше API-контроллер был обычным контроллером:

class UserApiController extends Controller

Теперь он специальный:

class UserApiController extends ApiController

И вместо:

return $this->success([...]);
return $this->error(...);

пишем более API-понятно:

return $this->ok([...]);
return $this->fail(...);
return $this->validationError([...]);
return $this->notFound(...);

Главная мысль:

Controller — для HTML-страниц.
ApiController — для JSON API.
Service и Model можно использовать и там, и там.

Дальше можно сделать единый формат API-ответов и обработку исключений API, чтобы любая ошибка в /api/... автоматически возвращалась JSON, а не HTML.