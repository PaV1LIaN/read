Идём дальше. Сейчас сделаем защиту от белых экранов — общий обработчик ошибок.

Зачем это нужно

Сейчас если где-то в проекте будет ошибка:

не найден класс
ошибка в SQL
ошибка в PHP
не найден файл view

может появиться белый экран или непонятная ошибка.

Мы хотим, чтобы фреймворк сам показывал нормальную страницу:

500
Ошибка приложения

Файл:
Строка:
Сообщение:

А если это JSON-запрос — возвращал JSON.


---

1. Создай /local/mvc/Core/ErrorHandler.php

<?php

namespace Local\Mvc\Core;

use Throwable;

/**
 * ErrorHandler
 *
 * Общий обработчик ошибок MVC.
 *
 * Его задача:
 * вместо белого экрана показать понятную ошибку.
 */
class ErrorHandler
{
    private static ?Request $request = null;

    public static function register(?Request $request = null): void
    {
        self::$request = $request;

        /**
         * Обычные PHP-ошибки превращаем в исключения.
         */
        set_error_handler(function ($severity, $message, $file, $line) {
            if (!(error_reporting() & $severity)) {
                return false;
            }

            throw new \ErrorException($message, 0, $severity, $file, $line);
        });

        /**
         * Исключения ловим здесь.
         */
        set_exception_handler(function (Throwable $e) {
            self::renderThrowable($e);
        });

        /**
         * Фатальные ошибки ловим в конце выполнения.
         */
        register_shutdown_function(function () {
            $error = error_get_last();

            if ($error === null) {
                return;
            }

            $fatalTypes = [
                E_ERROR,
                E_PARSE,
                E_CORE_ERROR,
                E_COMPILE_ERROR,
            ];

            if (!in_array($error['type'], $fatalTypes, true)) {
                return;
            }

            self::renderFatal($error);
        });
    }

    public static function renderThrowable(Throwable $e): void
    {
        self::log($e->getMessage(), $e->getFile(), $e->getLine());

        if (self::wantsJson()) {
            Response::json([
                'ok' => false,
                'error' => 'SERVER_ERROR',
                'details' => self::debugEnabled()
                    ? [
                        'message' => $e->getMessage(),
                        'file' => $e->getFile(),
                        'line' => $e->getLine(),
                    ]
                    : [
                        'message' => 'Внутренняя ошибка сервера',
                    ],
            ], 500)->send();

            return;
        }

        Response::html(self::errorHtml(
            'Ошибка приложения',
            $e->getMessage(),
            $e->getFile(),
            $e->getLine(),
            $e->getTraceAsString()
        ), 500)->send();
    }

    private static function renderFatal(array $error): void
    {
        $message = (string)($error['message'] ?? 'Fatal error');
        $file = (string)($error['file'] ?? '');
        $line = (int)($error['line'] ?? 0);

        self::log($message, $file, $line);

        if (self::wantsJson()) {
            Response::json([
                'ok' => false,
                'error' => 'FATAL_ERROR',
                'details' => self::debugEnabled()
                    ? [
                        'message' => $message,
                        'file' => $file,
                        'line' => $line,
                    ]
                    : [
                        'message' => 'Критическая ошибка сервера',
                    ],
            ], 500)->send();

            return;
        }

        Response::html(self::errorHtml(
            'Критическая ошибка',
            $message,
            $file,
            $line,
            ''
        ), 500)->send();
    }

    private static function errorHtml(string $title, string $message, string $file, int $line, string $trace): string
    {
        if (!self::debugEnabled()) {
            return '
                <div style="max-width:900px;margin:40px auto;padding:24px;border:1px solid #fecaca;border-radius:16px;background:#fef2f2;color:#991b1b;">
                    <h1 style="margin-top:0;">500</h1>
                    <p>Внутренняя ошибка сервера.</p>
                </div>
            ';
        }

        return '
            <div style="max-width:1100px;margin:40px auto;padding:24px;border:1px solid #fecaca;border-radius:16px;background:#fef2f2;color:#111827;font-family:Arial,sans-serif;">
                <h1 style="margin-top:0;color:#991b1b;">500 — ' . htmlspecialchars($title, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') . '</h1>

                <p><b>Сообщение:</b></p>
                <pre style="white-space:pre-wrap;background:#fff;padding:16px;border-radius:10px;border:1px solid #fecaca;">' . htmlspecialchars($message, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') . '</pre>

                <p><b>Файл:</b></p>
                <pre style="white-space:pre-wrap;background:#fff;padding:16px;border-radius:10px;border:1px solid #fecaca;">' . htmlspecialchars($file, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') . ':' . (int)$line . '</pre>

                ' . ($trace !== '' ? '
                    <p><b>Trace:</b></p>
                    <pre style="white-space:pre-wrap;background:#111827;color:#e5e7eb;padding:16px;border-radius:10px;overflow:auto;">' . htmlspecialchars($trace, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') . '</pre>
                ' : '') . '
            </div>
        ';
    }

    private static function wantsJson(): bool
    {
        if (!(self::$request instanceof Request)) {
            return false;
        }

        $accept = (string)self::$request->server('HTTP_ACCEPT', '');
        $path = self::$request->path();

        return str_contains($accept, 'application/json')
            || str_starts_with($path, '/api')
            || str_contains($path, '/ping');
    }

    private static function debugEnabled(): bool
    {
        return defined('LOCAL_MVC_DEBUG') && LOCAL_MVC_DEBUG === true;
    }

    private static function log(string $message, string $file, int $line): void
    {
        error_log('[LOCAL_MVC_ERROR] ' . $message . ' in ' . $file . ':' . $line);
    }
}


---

2. Обнови /local/mvc/Core/App.php

Полностью замени файл:

<?php

namespace Local\Mvc\Core;

/**
 * App
 *
 * Запускатель MVC-приложения.
 */
class App
{
    public static function run(?string $routesFile = null): void
    {
        $projectRoot = self::projectRoot();

        if ($routesFile === null) {
            $routesFile = $projectRoot . '/routes.php';
        }

        /**
         * Создаём Request как можно раньше,
         * чтобы ErrorHandler понимал текущий маршрут.
         */
        $request = Request::createFromGlobals();

        /**
         * Включаем общий обработчик ошибок.
         */
        ErrorHandler::register($request);

        try {
            if (!is_file($routesFile)) {
                Response::html(
                    '<h1>500</h1><p>Файл маршрутов не найден.</p><pre>'
                    . htmlspecialchars($routesFile)
                    . '</pre>',
                    500
                )->send();

                return;
            }

            $router = new Router();

            require $routesFile;

            $router->dispatch($request);
        } catch (\Throwable $e) {
            ErrorHandler::renderThrowable($e);
        }
    }

    public static function projectRoot(): string
    {
        if (!defined('LOCAL_MVC_PROJECT_ROOT')) {
            return $_SERVER['DOCUMENT_ROOT'] . '/local/mvc';
        }

        return rtrim((string)LOCAL_MVC_PROJECT_ROOT, '/');
    }

    public static function projectUrl(): string
    {
        if (!defined('LOCAL_MVC_PROJECT_URL')) {
            return '/local/mvc';
        }

        return rtrim((string)LOCAL_MVC_PROJECT_URL, '/');
    }

    public static function projectNamespace(): string
    {
        if (!defined('LOCAL_MVC_PROJECT_NAMESPACE')) {
            return 'Local\\Mvc\\';
        }

        return rtrim((string)LOCAL_MVC_PROJECT_NAMESPACE, '\\') . '\\';
    }
}


---

3. Включи debug в /local/mvc_demo/index.php

В файле:

/local/mvc_demo/index.php

добавь константу:

define('LOCAL_MVC_DEBUG', true);

Должно быть так:

<?php

define('LOCAL_MVC_PROJECT_ROOT', __DIR__);
define('LOCAL_MVC_PROJECT_URL', '/local/mvc_demo');
define('LOCAL_MVC_PROJECT_NAMESPACE', 'Local\\MvcDemo\\');

/**
 * Пока учимся — debug включён.
 * На боевом проекте лучше поставить false.
 */
define('LOCAL_MVC_DEBUG', true);

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/mvc/bootstrap.php';

use Local\Mvc\Core\App;

App::run();


---

4. Проверяем ошибку специально

Открой:

/local/mvc_demo/Controllers/HomeController.php

И временно в метод ping() добавь ошибку:

public function ping(): Response
{
    throw new \RuntimeException('Тестовая ошибка MVC');

    return $this->success([
        'message' => 'pong',
    ]);
}

Теперь открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/ping

Должен вернуться JSON:

{
  "ok": false,
  "error": "SERVER_ERROR",
  "details": {
    "message": "Тестовая ошибка MVC",
    "file": "...",
    "line": 123
  }
}

Потом обязательно убери строку:

throw new \RuntimeException('Тестовая ошибка MVC');


---

Что мы сделали

Раньше при ошибке мог быть белый экран.

Теперь:

ошибка в PHP
  ↓
ErrorHandler ловит ошибку
  ↓
пишет в error_log
  ↓
показывает понятную страницу 500
  ↓
для JSON-маршрутов возвращает JSON

Это очень важный кирпич фреймворка.

Следующим шагом можно сделать Logger, чтобы писать ошибки и события не только в error_log, а в отдельный файл проекта, например:

/local/mvc_demo/logs/app.log