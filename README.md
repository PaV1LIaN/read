Идём дальше.

Сейчас у нас маршруты открываются так:

/local/mvc/?route=/ping
/local/mvc/?route=/go-home

Это рабочий вариант, но некрасивый.

Мы хотим так:

/local/mvc/ping
/local/mvc/go-home
/local/mvc/admin/dashboard

Для этого нужно сделать 2 вещи:

1. Научить Request понимать красивый адрес.
2. Настроить веб-сервер, чтобы все красивые адреса попадали в /local/mvc/index.php.


---

Шаг 1. Обновляем Request.php

Заменяй файл:

/local/mvc/Core/Request.php

полностью на этот:

<?php

namespace Local\Mvc\Core;

/**
 * Request
 *
 * Класс для работы с запросом пользователя.
 */
class Request
{
    private array $get;
    private array $post;
    private array $server;
    private array $files;

    public function __construct(array $get, array $post, array $server, array $files = [])
    {
        $this->get = $get;
        $this->post = $post;
        $this->server = $server;
        $this->files = $files;
    }

    public static function createFromGlobals(): self
    {
        return new self($_GET, $_POST, $_SERVER, $_FILES);
    }

    public function get(string $key, mixed $default = null): mixed
    {
        return $this->get[$key] ?? $default;
    }

    public function post(string $key, mixed $default = null): mixed
    {
        return $this->post[$key] ?? $default;
    }

    public function input(string $key, mixed $default = null): mixed
    {
        if (array_key_exists($key, $this->post)) {
            return $this->post[$key];
        }

        if (array_key_exists($key, $this->get)) {
            return $this->get[$key];
        }

        return $default;
    }

    public function allGet(): array
    {
        return $this->get;
    }

    public function allPost(): array
    {
        return $this->post;
    }

    public function method(): string
    {
        return strtoupper((string)($this->server['REQUEST_METHOD'] ?? 'GET'));
    }

    public function isGet(): bool
    {
        return $this->method() === 'GET';
    }

    public function isPost(): bool
    {
        return $this->method() === 'POST';
    }

    /**
     * Получить путь маршрута.
     *
     * Поддерживает 2 варианта:
     *
     * Старый:
     * /local/mvc/?route=/ping
     *
     * Новый:
     * /local/mvc/ping
     */
    public function path(): string
    {
        /**
         * 1. Сначала проверяем старый вариант через ?route=
         *
         * Это нужно, чтобы у нас осталась запасная дверь.
         */
        $routeFromGet = trim((string)$this->get('route', ''));

        if ($routeFromGet !== '') {
            return $this->normalizePath($routeFromGet);
        }

        /**
         * 2. Если route нет, берём настоящий адрес из REQUEST_URI.
         *
         * Например:
         * /local/mvc/ping?x=1
         */
        $requestUri = (string)$this->server('REQUEST_URI', '/');

        /**
         * Убираем query string.
         *
         * Было:
         * /local/mvc/ping?x=1
         *
         * Стало:
         * /local/mvc/ping
         */
        $uriPath = parse_url($requestUri, PHP_URL_PATH);

        if (!is_string($uriPath) || $uriPath === '') {
            $uriPath = '/';
        }

        /**
         * SCRIPT_NAME обычно такой:
         * /local/mvc/index.php
         *
         * Нам нужно получить базовую папку:
         * /local/mvc
         */
        $scriptName = (string)$this->server('SCRIPT_NAME', '');
        $scriptDir = rtrim(str_replace('\\', '/', dirname($scriptName)), '/');

        /**
         * Если сайт открыт через /local/mvc/ping,
         * то убираем начало /local/mvc.
         *
         * Было:
         * /local/mvc/ping
         *
         * Стало:
         * /ping
         */
        if ($scriptDir !== '' && $scriptDir !== '/' && str_starts_with($uriPath, $scriptDir)) {
            $uriPath = substr($uriPath, strlen($scriptDir));
        }

        /**
         * Если вдруг в адресе есть index.php:
         *
         * /local/mvc/index.php/ping
         *
         * уберём его.
         */
        if (str_starts_with($uriPath, '/index.php')) {
            $uriPath = substr($uriPath, strlen('/index.php'));
        }

        return $this->normalizePath($uriPath);
    }

    public function file(string $key): ?array
    {
        return $this->files[$key] ?? null;
    }

    public function server(string $key, mixed $default = null): mixed
    {
        return $this->server[$key] ?? $default;
    }

    /**
     * Привести путь к нормальному виду.
     *
     * ''       => '/'
     * 'ping'  => '/ping'
     * '/ping' => '/ping'
     * '/ping/' => '/ping'
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
}

Что мы сделали простыми словами

Раньше Request понимал только это:

?route=/ping

Теперь он понимает и это:

/local/mvc/ping

Но важно: сам PHP увидит /local/mvc/ping только если веб-сервер отправит этот адрес в /local/mvc/index.php.


---

Шаг 2. Проверяем старый вариант

Сначала проверь, что мы ничего не сломали:

https://bitrix24-stage.gaz.ru/local/mvc/?route=/ping

Должен вернуться JSON:

{
  "ok": true,
  "data": {
    "message": "pong",
    "mvc": true,
    "method": "GET",
    "path": "/ping"
  }
}

Если старый вариант работает — значит Request.php обновился нормально.


---

Шаг 3. Пробуем красивый адрес

Теперь попробуй:

https://bitrix24-stage.gaz.ru/local/mvc/ping

Тут возможны 2 варианта.

Вариант А — сразу работает

Если вернулся JSON — отлично, значит веб-сервер уже правильно прокидывает запросы.

Вариант Б — 404 или 500 от сервера

Это тоже нормально.

Почему?

Потому что веб-сервер думает:

Пользователь просит файл или папку /local/mvc/ping.
Такого файла нет.
Значит 404.

Нужно объяснить серверу:

Если файла нет внутри /local/mvc/,
отдай запрос в /local/mvc/index.php.


---

Шаг 4. Если работает Apache — добавь .htaccess

Создай файл:

/local/mvc/.htaccess

Код:

Options -Indexes

<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteBase /local/mvc/

    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteCond %{REQUEST_FILENAME} !-d

    RewriteRule ^ index.php [L,QSA]
</IfModule>

После этого снова проверь:

https://bitrix24-stage.gaz.ru/local/mvc/ping


---

Шаг 5. Если у тебя Angie/Nginx

На BitrixVM часто стоит nginx или angie, и .htaccess может вообще не читаться.

Тогда правило надо добавлять в конфиг сайта на уровне веб-сервера.

Пример для nginx/angie:

location ^~ /local/mvc/ {
    try_files $uri $uri/ /local/mvc/index.php?$query_string;
}

После изменения конфига нужно проверить и перезагрузить веб-сервер:

sudo nginx -t
sudo systemctl reload nginx

Если у тебя именно angie, команды могут быть такими:

sudo angie -t
sudo systemctl reload angie

Смысл правила очень простой:

Сначала попробуй найти настоящий файл.
Потом попробуй найти настоящую папку.
Если не нашёл — отправь всё в /local/mvc/index.php.


---

Шаг 6. Добавим тестовую страницу с красивым адресом

Создай новый метод в:

/local/mvc/Controllers/HomeController.php

Внутрь класса HomeController добавь метод:

public function about(): Response
{
    return $this->render('home/index', [
        'title' => 'О нашем MVC',
        'message' => 'Это страница /about. Красивые маршруты работают.',
    ]);
}

Полный HomeController.php может быть таким:

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
    public function index(): Response
    {
        $name = (string)$this->request->get('name', 'Гость');

        return $this->render('home/index', [
            'title' => 'MVC работает',
            'message' => 'Привет, ' . $name . '! Красивые маршруты почти готовы.',
        ]);
    }

    public function ping(): Response
    {
        return $this->success([
            'message' => 'pong',
            'mvc' => true,
            'method' => $this->request->method(),
            'path' => $this->request->path(),
        ]);
    }

    public function goHome(): Response
    {
        return $this->redirect('/local/mvc/');
    }

    public function about(): Response
    {
        return $this->render('home/index', [
            'title' => 'О нашем MVC',
            'message' => 'Это страница /about. Красивые маршруты работают.',
        ]);
    }
}


---

Шаг 7. Обновляем routes.php

Заменяй:

/local/mvc/routes.php

на:

<?php

use Local\Mvc\Core\Router;
use Local\Mvc\Controllers\HomeController;

/** @var Router $router */

$router->get('/', [HomeController::class, 'index']);

$router->get('/ping', [HomeController::class, 'ping']);

$router->get('/go-home', [HomeController::class, 'goHome']);

$router->get('/about', [HomeController::class, 'about']);


---

Шаг 8. Проверяем

Старый вариант:

https://bitrix24-stage.gaz.ru/local/mvc/?route=/about

Новый красивый вариант:

https://bitrix24-stage.gaz.ru/local/mvc/about

Оба должны показать страницу:

О нашем MVC
Это страница /about. Красивые маршруты работают.


---

Что у нас получилось

Теперь наш MVC умеет два режима.

Старый безопасный режим:

/local/mvc/?route=/ping

Красивый режим:

/local/mvc/ping

Путь запроса теперь такой

Браузер открыл /local/mvc/about
  ↓
Веб-сервер отправил запрос в /local/mvc/index.php
  ↓
index.php создал Request
  ↓
Request понял, что путь = /about
  ↓
Router нашёл маршрут /about
  ↓
HomeController::about()
  ↓
View показал HTML


---

Главное, что надо понять

Красивые URL работают не только в PHP.

Тут участвуют 2 стороны:

1. Веб-сервер
   Он должен отправить /local/mvc/about в index.php.

2. Наш Request
   Он должен понять, что настоящий маршрут — /about.

Если старый вариант работает, а красивый нет — проблема не в PHP-коде MVC, а в настройке веб-сервера.

Следующим шагом сделаем Layout, чтобы у нас был свой общий шаблон MVC: меню, контейнер, стили, заголовок, кнопки. Тогда страницы будут выглядеть не как случайный HTML, а как нормальная маленькая админка.