Идём дальше. Сделаем Laravel-like helper-функции:

request()
response()
redirect()

Чтобы можно было писать почти как в Laravel:

$name = request('name');

return response()->json([
    'ok' => true,
]);

return redirect()->route('form.index');


---

1. Создай /local/mvc/Core/ResponseFactory.php

<?php

namespace Local\Mvc\Core;

/**
 * ResponseFactory
 *
 * Laravel-like фабрика ответов.
 *
 * Позволяет писать:
 * response()->json(...)
 * response()->html(...)
 * response()->redirect(...)
 */
class ResponseFactory
{
    public function html(string $content = '', int $status = 200): Response
    {
        return Response::html($content, $status);
    }

    public function json(array $data = [], int $status = 200): Response
    {
        return Response::json($data, $status);
    }

    public function redirect(string $url, int $status = 302): Response
    {
        return Response::redirect($url, $status);
    }

    public function noContent(int $status = 204): Response
    {
        return new Response('', $status);
    }
}


---

2. Создай /local/mvc/Core/Redirector.php

<?php

namespace Local\Mvc\Core;

/**
 * Redirector
 *
 * Laravel-like помощник для редиректов.
 *
 * Позволяет писать:
 * redirect()->to('/some/url')
 * redirect()->route('form.index')
 * redirect()->back()
 */
class Redirector
{
    public function __construct(
        private Request $request
    ) {}

    public function to(string $url, int $status = 302): Response
    {
        return Response::redirect($url, $status);
    }

    public function route(string $name, array $params = [], array $query = [], int $status = 302): Response
    {
        return Response::redirect(App::route($name, $params, $query), $status);
    }

    public function back(string $fallback = '/', int $status = 302): Response
    {
        $referer = (string)$this->request->server('HTTP_REFERER', '');

        if ($this->isSafeRedirectUrl($referer)) {
            return Response::redirect($referer, $status);
        }

        return Response::redirect($this->projectUrlPath($fallback), $status);
    }

    private function projectUrlPath(string $path = '/'): string
    {
        $base = App::projectUrl();

        $path = trim($path);

        if ($path === '' || $path === '/') {
            return $base . '/';
        }

        return $base . '/' . trim($path, '/');
    }

    private function isSafeRedirectUrl(string $url): bool
    {
        $url = trim($url);

        if ($url === '') {
            return false;
        }

        if (str_starts_with($url, '/')) {
            return true;
        }

        $currentHost = (string)$this->request->server('HTTP_HOST', '');

        $parts = parse_url($url);

        if (!is_array($parts)) {
            return false;
        }

        $urlHost = (string)($parts['host'] ?? '');

        if ($urlHost === '' || $currentHost === '') {
            return false;
        }

        return strcasecmp($urlHost, $currentHost) === 0;
    }
}


---

3. Обнови /local/mvc/Core/Request.php

Добавим Laravel-like методы input() и all().

Внутрь класса Request добавь:

/**
 * Получить значение из запроса.
 *
 * Порядок:
 * 1. POST
 * 2. JSON body
 * 3. GET
 */
public function input(string $key, mixed $default = null): mixed
{
    if (array_key_exists($key, $this->postAll())) {
        return $this->postAll()[$key];
    }

    if (array_key_exists($key, $this->jsonAll())) {
        return $this->jsonAll()[$key];
    }

    if (array_key_exists($key, $this->getAll())) {
        return $this->getAll()[$key];
    }

    return $default;
}

/**
 * Все GET-данные.
 */
public function getAll(): array
{
    return $this->get;
}

/**
 * Все данные запроса.
 */
public function all(): array
{
    return array_merge(
        $this->getAll(),
        $this->postAll(),
        $this->jsonAll()
    );
}

У тебя уже должен быть метод:

public function postAll(): array
{
    return $this->post;
}

Если его нет — тоже добавь.


---

4. Обнови /local/mvc/Core/App.php

В методе run() найди место:

$container->instance(Request::class, $request);
$container->instance(Router::class, $router);
$container->instance(Container::class, $container);

$container->singleton(\Local\Mvc\Core\LogManager::class, \Local\Mvc\Core\LogManager::class);
$container->singleton(\Local\Mvc\Core\ConfigManager::class, \Local\Mvc\Core\ConfigManager::class);

Сразу после этого добавь:

$container->singleton(\Local\Mvc\Core\ResponseFactory::class, \Local\Mvc\Core\ResponseFactory::class);
$container->singleton(\Local\Mvc\Core\Redirector::class, \Local\Mvc\Core\Redirector::class);

Итог:

$container->instance(Request::class, $request);
$container->instance(Router::class, $router);
$container->instance(Container::class, $container);

$container->singleton(\Local\Mvc\Core\LogManager::class, \Local\Mvc\Core\LogManager::class);
$container->singleton(\Local\Mvc\Core\ConfigManager::class, \Local\Mvc\Core\ConfigManager::class);
$container->singleton(\Local\Mvc\Core\ResponseFactory::class, \Local\Mvc\Core\ResponseFactory::class);
$container->singleton(\Local\Mvc\Core\Redirector::class, \Local\Mvc\Core\Redirector::class);


---

5. Обнови /local/mvc/helpers.php

В конец файла добавь:

if (!function_exists('request')) {
    /**
     * Laravel-like request()
     *
     * request() вернёт объект Request.
     * request('name') вернёт значение поля name.
     */
    function request(?string $key = null, mixed $default = null): mixed
    {
        $request = \Local\Mvc\Core\App::make(\Local\Mvc\Core\Request::class);

        if ($key === null) {
            return $request;
        }

        return $request->input($key, $default);
    }
}

if (!function_exists('response')) {
    /**
     * Laravel-like response()
     *
     * response() вернёт ResponseFactory.
     * response('text') вернёт HTML response.
     */
    function response(?string $content = null, int $status = 200): mixed
    {
        $factory = \Local\Mvc\Core\App::make(\Local\Mvc\Core\ResponseFactory::class);

        if ($content === null) {
            return $factory;
        }

        return $factory->html($content, $status);
    }
}

if (!function_exists('redirect')) {
    /**
     * Laravel-like redirect()
     *
     * redirect() вернёт Redirector.
     * redirect('/url') сразу сделает redirect.
     */
    function redirect(?string $to = null): mixed
    {
        $redirector = \Local\Mvc\Core\App::make(\Local\Mvc\Core\Redirector::class);

        if ($to === null) {
            return $redirector;
        }

        return $redirector->to($to);
    }
}


---

6. Добавь тест в HomeController

Открой:

/local/mvc_demo/Controllers/HomeController.php

Добавь метод:

public function helperTest(): Response
{
    return response()->json([
        'ok' => true,
        'data' => [
            'message' => 'Laravel-like helpers работают',
            'app_name' => config('app.name'),
            'request_path' => request()->path(),
            'q' => request('q', ''),
        ],
    ]);
}


---

7. Добавь маршрут в /local/mvc_demo/routes.php

Route::get('/helper-test', [HomeController::class, 'helperTest'])
    ->name('helper.test');


---

8. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/helper-test?q=hello

Должен быть JSON:

{
  "ok": true,
  "data": {
    "message": "Laravel-like helpers работают",
    "app_name": "MVC Demo",
    "request_path": "/helper-test",
    "q": "hello"
  }
}


---

Что мы сделали

Теперь можно писать ближе к Laravel:

request()
request('q')
response()->json([...])
redirect()->route('form.index')
redirect()->back()
config('app.name')
route('form.index')
old('name')
csrf_field()
method_field('DELETE')

Раньше было так:

return $this->success([
    'message' => 'ok',
]);

Теперь можно и так:

return response()->json([
    'ok' => true,
    'data' => [
        'message' => 'ok',
    ],
]);

Главная мысль:

helpers — это короткий Laravel-like доступ к основным частям фреймворка.

Дальше логичный Laravel-like шаг — сделать Model query builder, чтобы вместо ручного SQL постепенно прийти к стилю:

User::query()
    ->where('ACTIVE', 'Y')
    ->orderBy('ID', 'desc')
    ->paginate(10);