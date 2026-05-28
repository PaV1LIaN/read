Идём дальше. Сейчас сделаем единые API-ошибки через исключения.

Зачем это нужно:

Сейчас API-контроллер сам возвращает ошибку:
return $this->notFound(...)

А хочется уметь так:
throw new HttpException(...)

Тогда если в API что-то сломалось, фреймворк сам вернёт JSON:

{
  "ok": false,
  "error": "USER_NOT_FOUND",
  "details": {
    "message": "Пользователь не найден"
  }
}

А не HTML-страницу с ошибкой.


---

1. Создай /local/mvc/Core/HttpException.php

<?php

namespace Local\Mvc\Core;

use RuntimeException;

/**
 * HttpException
 *
 * Исключение с HTTP-статусом.
 *
 * Например:
 * 404 — не найдено
 * 403 — доступ запрещён
 * 422 — ошибка валидации
 * 500 — ошибка сервера
 */
class HttpException extends RuntimeException
{
    private int $status;
    private string $error;
    private array $details;

    public function __construct(
        int $status,
        string $error,
        string $message = '',
        array $details = []
    ) {
        parent::__construct($message);

        $this->status = $status;
        $this->error = $error;
        $this->details = $details;
    }

    public function status(): int
    {
        return $this->status;
    }

    public function error(): string
    {
        return $this->error;
    }

    public function details(): array
    {
        return $this->details;
    }
}


---

2. Обнови /local/mvc/Core/ErrorHandler.php

Найди метод:

public static function renderThrowable(Throwable $e): void

И замени его полностью на этот:

public static function renderThrowable(Throwable $e): void
{
    self::log($e->getMessage(), $e->getFile(), $e->getLine());

    $status = 500;
    $error = 'SERVER_ERROR';
    $message = 'Внутренняя ошибка сервера';
    $details = [];

    /**
     * Если это наше HTTP-исключение,
     * берём статус и код ошибки из него.
     */
    if ($e instanceof HttpException) {
        $status = $e->status();
        $error = $e->error();
        $message = $e->getMessage() !== '' ? $e->getMessage() : 'Ошибка запроса';
        $details = $e->details();
    } else {
        $message = $e->getMessage();
    }

    if (self::wantsJson()) {
        $responseDetails = array_merge([
            'message' => self::debugEnabled() || $e instanceof HttpException
                ? $message
                : 'Внутренняя ошибка сервера',
        ], $details);

        if (self::debugEnabled() && !($e instanceof HttpException)) {
            $responseDetails['file'] = $e->getFile();
            $responseDetails['line'] = $e->getLine();
        }

        Response::json([
            'ok' => false,
            'error' => $error,
            'details' => $responseDetails,
        ], $status)->send();

        return;
    }

    Response::html(self::errorHtml(
        'Ошибка приложения',
        $message,
        $e->getFile(),
        $e->getLine(),
        $e->getTraceAsString()
    ), $status)->send();
}

Что изменилось:

Если ошибка обычная — SERVER_ERROR.
Если ошибка HttpException — берём её status/error/details.


---

3. Обнови /local/mvc/Core/ApiController.php

Полностью замени файл:

<?php

namespace Local\Mvc\Core;

/**
 * ApiController
 *
 * Базовый контроллер для API.
 */
class ApiController extends Controller
{
    protected function ok(array $data = [], int $status = 200): Response
    {
        return $this->json([
            'ok' => true,
            'data' => $data,
        ], $status);
    }

    protected function fail(string $error, array $details = [], int $status = 400): Response
    {
        return $this->json([
            'ok' => false,
            'error' => $error,
            'details' => $details,
        ], $status);
    }

    protected function validationError(array $errors): Response
    {
        return $this->fail('VALIDATION_ERROR', [
            'errors' => $errors,
        ], 422);
    }

    protected function notFound(string $message = 'Запись не найдена', array $details = []): Response
    {
        return $this->fail('NOT_FOUND', array_merge([
            'message' => $message,
        ], $details), 404);
    }

    protected function forbidden(string $message = 'Доступ запрещён'): Response
    {
        return $this->fail('FORBIDDEN', [
            'message' => $message,
        ], 403);
    }

    protected function jsonData(): array
    {
        return $this->request->jsonAll();
    }

    /**
     * Выбросить API-ошибку.
     *
     * Она будет поймана ErrorHandler,
     * и клиент получит JSON.
     */
    protected function abort(
        int $status,
        string $error,
        string $message = '',
        array $details = []
    ): void {
        throw new HttpException($status, $error, $message, $details);
    }

    protected function abortNotFound(string $message = 'Запись не найдена', array $details = []): void
    {
        $this->abort(404, 'NOT_FOUND', $message, $details);
    }

    protected function abortForbidden(string $message = 'Доступ запрещён'): void
    {
        $this->abort(403, 'FORBIDDEN', $message);
    }

    protected function abortValidation(array $errors): void
    {
        $this->abort(422, 'VALIDATION_ERROR', 'Ошибка валидации', [
            'errors' => $errors,
        ]);
    }
}


---

4. Обнови /local/mvc_demo/Controllers/UserApiController.php

Сделаем пример: если пользователя нет, не возвращаем return $this->notFound(...), а выбрасываем исключение.

Замени метод show() на:

public function show(string $id): Response
{
    $userService = new UserService();

    $user = $userService->findForDetail((int)$id);

    if (!$user) {
        $this->abortNotFound('Пользователь не найден', [
            'id' => (int)$id,
        ]);
    }

    return $this->ok([
        'user' => $user,
    ]);
}


---

5. Добавим тестовую API-ошибку

Открой:

/local/mvc_demo/Controllers/AjaxDemoController.php

Внутрь класса добавь метод:

public function errorTest(): Response
{
    $this->abort(418, 'TEST_API_EXCEPTION', 'Это тестовая API-ошибка', [
        'hint' => 'Так мы проверяем HttpException',
    ]);

    return $this->ok();
}


---

6. Добавь маршрут в /local/mvc_demo/routes.php

В API-группу добавь:

$router->get('/error-test', [AjaxDemoController::class, 'errorTest']);

Должно быть примерно так:

$router->group([
    'prefix' => '/api',
    'middleware' => ['auth', 'admin'],
], function (Router $router) {
    $router->get('/users', [UserApiController::class, 'index']);
    $router->get('/users/{id:\d+}', [UserApiController::class, 'show']);

    $router->post('/ajax-demo/echo', [AjaxDemoController::class, 'echoText'], ['csrf']);

    $router->get('/error-test', [AjaxDemoController::class, 'errorTest']);
});


---

7. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Проверь тестовую ошибку:

https://bitrix24-stage.gaz.ru/local/mvc_demo/api/error-test

Должен быть JSON:

{
  "ok": false,
  "error": "TEST_API_EXCEPTION",
  "details": {
    "message": "Это тестовая API-ошибка",
    "hint": "Так мы проверяем HttpException"
  }
}

Проверь несуществующего пользователя:

https://bitrix24-stage.gaz.ru/local/mvc_demo/api/users/999999999

Должно быть:

{
  "ok": false,
  "error": "NOT_FOUND",
  "details": {
    "message": "Пользователь не найден",
    "id": 999999999
  }
}


---

Что мы сделали

Теперь API может работать так:

$this->abortNotFound('Пользователь не найден');
$this->abortForbidden('Нет доступа');
$this->abortValidation([
    'name' => ['Введите имя']
]);

А ErrorHandler сам превратит это в JSON.

Главная мысль:

Контроллер может не собирать ошибку вручную.
Он может просто выбросить HttpException.
Фреймворк сам отдаст правильный JSON и HTTP-статус.

Это пригодится в реальных API:

создание сайта
удаление страницы
загрузка файла
смена статуса заявки
обновление настроек