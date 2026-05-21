Отлично, идём дальше.

Сейчас у нас уже есть:

Request  — что пришло от пользователя
Router   — куда отправить пользователя
Controller — кто обрабатывает запрос
View     — что показать

Теперь добавим:

Response — что вернуть пользователю

Зачем нужен Response

Сейчас контроллер сам сразу делает:

echo json_encode(...);
exit;

или подключает HTML.

Пока это нормально. Но дальше будет неудобно.

Мы хотим, чтобы контроллер говорил:

return $this->success([...]);

или:

return $this->render('home/index', [...]);

или:

return $this->redirect('/local/mvc/');

То есть контроллер не должен сам напрямую “кричать в браузер”.
Он должен подготовить ответ, а Router уже отправит его пользователю.

Представь так:

Контроллер приготовил письмо.
Router отнёс письмо пользователю.


---

Шаг 1. Создаём Response.php

Создай файл:

/local/mvc/Core/Response.php

Полный код:

<?php

namespace Local\Mvc\Core;

/**
 * Response
 *
 * Это ответ сервера пользователю.
 *
 * Он может быть:
 * - HTML-страницей
 * - JSON-ответом
 * - редиректом
 * - ошибкой
 */
class Response
{
    private string $content;
    private int $status;
    private array $headers;

    public function __construct(string $content = '', int $status = 200, array $headers = [])
    {
        $this->content = $content;
        $this->status = $status;
        $this->headers = $headers;
    }

    /**
     * HTML-ответ.
     */
    public static function html(string $content, int $status = 200): self
    {
        return new self($content, $status, [
            'Content-Type' => 'text/html; charset=utf-8',
        ]);
    }

    /**
     * JSON-ответ.
     */
    public static function json(array $data, int $status = 200): self
    {
        return new self(
            json_encode($data, JSON_UNESCAPED_UNICODE),
            $status,
            [
                'Content-Type' => 'application/json; charset=utf-8',
            ]
        );
    }

    /**
     * Редирект.
     *
     * Например:
     * return $this->redirect('/local/mvc/');
     */
    public static function redirect(string $url, int $status = 302): self
    {
        return new self('', $status, [
            'Location' => $url,
        ]);
    }

    /**
     * Добавить заголовок.
     */
    public function header(string $name, string $value): self
    {
        $this->headers[$name] = $value;

        return $this;
    }

    /**
     * Отправить ответ пользователю.
     */
    public function send(): void
    {
        if (!headers_sent()) {
            http_response_code($this->status);

            foreach ($this->headers as $name => $value) {
                header($name . ': ' . $value, true);
            }
        }

        echo $this->content;
    }
}


---

Шаг 2. Обновляем Controller.php

Теперь методы render(), success(), error() будут не сразу выводить ответ, а возвращать объект Response.

Замени файл:

/local/mvc/Core/Controller.php

полностью:

<?php

namespace Local\Mvc\Core;

/**
 * Controller
 *
 * Базовый контроллер.
 *
 * Все остальные контроллеры наследуются от него.
 */
class Controller
{
    /**
     * Текущий запрос.
     */
    protected Request $request;

    public function __construct(?Request $request = null)
    {
        $this->request = $request ?? Request::createFromGlobals();
    }

    /**
     * Показать HTML-страницу.
     *
     * Теперь этот метод возвращает Response.
     */
    protected function render(string $view, array $params = []): Response
    {
        $viewFile = dirname(__DIR__) . '/Views/' . $view . '.php';

        if (!is_file($viewFile)) {
            return Response::html(
                '<h1>500</h1><p>View не найден.</p><pre>' . htmlspecialchars($viewFile) . '</pre>',
                500
            );
        }

        /**
         * extract превращает массив в переменные.
         *
         * Например:
         * ['title' => 'Главная']
         *
         * станет:
         * $title = 'Главная';
         */
        extract($params);

        /**
         * Включаем буфер.
         *
         * Простыми словами:
         * PHP будет не сразу отправлять HTML в браузер,
         * а сначала сложит его во временную коробку.
         */
        ob_start();

        require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/header.php';

        require $viewFile;

        require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/footer.php';

        /**
         * Забираем всё, что попало в буфер.
         */
        $content = ob_get_clean();

        return Response::html($content);
    }

    /**
     * Вернуть произвольный JSON.
     */
    protected function json(array $data, int $status = 200): Response
    {
        return Response::json($data, $status);
    }

    /**
     * Успешный JSON-ответ.
     */
    protected function success(array $data = []): Response
    {
        return $this->json([
            'ok' => true,
            'data' => $data,
        ]);
    }

    /**
     * JSON-ошибка.
     */
    protected function error(string $message, array $details = [], int $status = 400): Response
    {
        return $this->json([
            'ok' => false,
            'error' => $message,
            'details' => $details,
        ], $status);
    }

    /**
     * Редирект.
     */
    protected function redirect(string $url): Response
    {
        return Response::redirect($url);
    }
}

Что изменилось

Раньше было:

$this->success([...]);
exit;

Теперь будет:

return $this->success([...]);

То есть контроллер возвращает ответ, а не завершает работу сам.


---

Шаг 3. Обновляем Router.php

Router теперь должен получить результат от контроллера и отправить его.

Замени файл:

/local/mvc/Core/Router.php

полностью:

<?php

namespace Local\Mvc\Core;

/**
 * Router
 *
 * Диспетчер маршрутов.
 */
class Router
{
    private array $routes = [];

    /**
     * Зарегистрировать GET-маршрут.
     */
    public function get(string $path, array $handler): void
    {
        $this->add('GET', $path, $handler);
    }

    /**
     * Зарегистрировать POST-маршрут.
     */
    public function post(string $path, array $handler): void
    {
        $this->add('POST', $path, $handler);
    }

    /**
     * Добавить маршрут.
     */
    private function add(string $method, string $path, array $handler): void
    {
        $method = strtoupper($method);
        $path = $this->normalizePath($path);

        $this->routes[$method][$path] = $handler;
    }

    /**
     * Запустить нужный контроллер по текущему запросу.
     */
    public function dispatch(Request $request): void
    {
        $method = $request->method();
        $path = $this->normalizePath($request->path());

        if (!isset($this->routes[$method][$path])) {
            $this->notFound($method, $path);
            return;
        }

        $handler = $this->routes[$method][$path];

        $controllerClass = $handler[0] ?? null;
        $controllerMethod = $handler[1] ?? null;

        if (!$controllerClass || !class_exists($controllerClass)) {
            $this->serverError('Контроллер не найден: ' . (string)$controllerClass);
            return;
        }

        /**
         * Создаём контроллер и передаём ему Request.
         */
        $controller = new $controllerClass($request);

        if (!$controllerMethod || !method_exists($controller, $controllerMethod)) {
            $methods = get_class_methods($controller);

            $this->serverError(
                'Метод контроллера не найден: ' . $controllerClass . '::' . (string)$controllerMethod
                . "\n\nPHP видит такие методы:\n"
                . implode("\n", $methods)
            );

            return;
        }

        /**
         * Вызываем метод контроллера.
         *
         * Теперь контроллер может вернуть Response.
         */
        $result = $controller->{$controllerMethod}();

        /**
         * Если контроллер вернул Response — отправляем его.
         */
        if ($result instanceof Response) {
            $result->send();
            return;
        }

        /**
         * Если контроллер ничего не вернул,
         * значит либо он сам всё вывел, либо метод написан неправильно.
         */
    }

    /**
     * Привести путь к нормальному виду.
     */
    private function normalizePath(string $path): string
    {
        $path = trim($path);

        if ($path === '') {
            return '/';
        }

        $path = '/' . trim($path, '/');

        if ($path !== '/') {
            $path = rtrim($path, '/');
        }

        return $path;
    }

    /**
     * 404 — маршрут не найден.
     */
    private function notFound(string $method, string $path): void
    {
        Response::html(
            '<h1>404</h1>'
            . '<p>Маршрут не найден.</p>'
            . '<pre>'
            . 'Method: ' . htmlspecialchars($method) . "\n"
            . 'Path: ' . htmlspecialchars($path) . "\n"
            . '</pre>',
            404
        )->send();

        exit;
    }

    /**
     * 500 — ошибка внутри MVC.
     */
    private function serverError(string $message): void
    {
        Response::html(
            '<h1>500</h1>'
            . '<p>Ошибка MVC.</p>'
            . '<pre>' . htmlspecialchars($message) . '</pre>',
            500
        )->send();

        exit;
    }
}


---

Шаг 4. Обновляем HomeController.php

Теперь методы должны возвращать Response.

Замени файл:

/local/mvc/Controllers/HomeController.php

полностью:

<?php

namespace Local\Mvc\Controllers;

use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Response;

/**
 * HomeController
 *
 * Тестовый контроллер.
 */
class HomeController extends Controller
{
    /**
     * Главная страница.
     */
    public function index(): Response
    {
        $name = (string)$this->request->get('name', 'Гость');

        return $this->render('home/index', [
            'title' => 'MVC работает',
            'message' => 'Привет, ' . $name . '! Response успешно работает.',
        ]);
    }

    /**
     * Тестовый JSON-ответ.
     */
    public function ping(): Response
    {
        return $this->success([
            'message' => 'pong',
            'mvc' => true,
            'method' => $this->request->method(),
            'path' => $this->request->path(),
        ]);
    }

    /**
     * Тест редиректа.
     */
    public function goHome(): Response
    {
        return $this->redirect('/local/mvc/');
    }
}


---

Шаг 5. Обновляем routes.php

Добавим маршрут для проверки редиректа.

Замени файл:

/local/mvc/routes.php

полностью:

<?php

use Local\Mvc\Core\Router;
use Local\Mvc\Controllers\HomeController;

/** @var Router $router */

$router->get('/', [HomeController::class, 'index']);

$router->get('/ping', [HomeController::class, 'ping']);

$router->get('/go-home', [HomeController::class, 'goHome']);


---

Шаг 6. Проверяем

Открой:

https://bitrix24-stage.gaz.ru/local/mvc/

Должно быть:

Привет, Гость! Response успешно работает.

Проверь с именем:

https://bitrix24-stage.gaz.ru/local/mvc/?name=Алексей

Должно быть:

Привет, Алексей! Response успешно работает.

Проверь JSON:

https://bitrix24-stage.gaz.ru/local/mvc/?route=/ping

Должно быть примерно:

{
    "ok": true,
    "data": {
        "message": "pong",
        "mvc": true,
        "method": "GET",
        "path": "/ping"
    }
}

Проверь редирект:

https://bitrix24-stage.gaz.ru/local/mvc/?route=/go-home

Он должен перекинуть обратно на:

/local/mvc/


---

Что мы сделали простыми словами

Раньше было так:

Контроллер сам сразу выводил результат в браузер.

Теперь так:

Контроллер готовит Response.
Router отправляет Response пользователю.

То есть:

Браузер
  ↓
index.php
  ↓
Request
  ↓
Router
  ↓
Controller
  ↓
Response
  ↓
Браузер

Почему это хорошо

Теперь у нас единый способ возвращать ответы:

return $this->render('home/index', []);

return $this->success(['id' => 5]);

return $this->error('Нет доступа');

return $this->redirect('/some/page/');

Это очень удобно, когда дальше пойдут:

формы
API
админка
ошибки
проверка прав
создание сайтов
сохранение страниц


---

Текущая структура

Теперь должно быть так:

/local/mvc/
  index.php
  bootstrap.php
  routes.php

  Core/
    Controller.php
    Request.php
    Response.php
    Router.php

  Controllers/
    HomeController.php

  Views/
    home/
      index.php

Следующим шагом сделаем красивые URL без ?route=, чтобы можно было открывать так:

/local/mvc/ping
/local/mvc/go-home
/local/mvc/admin/dashboard

А не так:

/local/mvc/?route=/ping