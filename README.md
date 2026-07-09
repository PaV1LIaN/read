Отлично. Тогда переходим в режим production-hardening. Первое, что надо довести до нормального уровня — ошибки 403 / 404 / 500.

Сейчас у тебя при ошибке показывается технический экран. Для production нужно так:

debug=true  → показываем подробности ошибки
debug=false → показываем красивую страницу без trace, путей и SQL


---

Шаг 1. Создай /local/mvc/Core/HttpException.php

Полный путь:

/srv/bx/docroot/local/mvc/Core/HttpException.php

Код:

<?php

namespace Local\Mvc\Core;

use RuntimeException;

class HttpException extends RuntimeException
{
    public function __construct(
        private int $statusCode = 500,
        string $message = '',
        private array $context = []
    ) {
        parent::__construct($message !== '' ? $message : self::defaultMessage($statusCode));
    }

    public function statusCode(): int
    {
        return $this->statusCode;
    }

    public function context(): array
    {
        return $this->context;
    }

    private static function defaultMessage(int $statusCode): string
    {
        return match ($statusCode) {
            400 => 'Некорректный запрос',
            401 => 'Необходима авторизация',
            403 => 'Доступ запрещён',
            404 => 'Страница не найдена',
            419 => 'Сессия истекла',
            422 => 'Ошибка валидации',
            default => 'Ошибка приложения',
        };
    }
}


---

Шаг 2. Замени /local/mvc/Core/ErrorHandler.php

Полный путь:

/srv/bx/docroot/local/mvc/Core/ErrorHandler.php

Полностью замени файл:

<?php

namespace Local\Mvc\Core;

use ErrorException;
use RuntimeException;
use Throwable;

class ErrorHandler
{
    public static function register(): void
    {
        set_error_handler(function (int $severity, string $message, string $file, int $line): bool {
            if (!(error_reporting() & $severity)) {
                return false;
            }

            throw new ErrorException($message, 0, $severity, $file, $line);
        });

        set_exception_handler(function (Throwable $e): void {
            self::renderThrowable($e);
        });
    }

    public static function renderThrowable(Throwable $e): void
    {
        self::logThrowable($e);

        if ($e instanceof ValidationException) {
            self::renderValidationException($e);
            return;
        }

        if ($e instanceof AuthorizationException) {
            self::renderErrorPage(
                403,
                'Доступ запрещён',
                'У вас нет прав для выполнения этого действия.',
                $e
            );
            return;
        }

        if ($e instanceof ModelNotFoundException) {
            self::renderErrorPage(
                404,
                'Запись не найдена',
                'Запрошенная запись не найдена или была удалена.',
                $e
            );
            return;
        }

        if ($e instanceof HttpException) {
            self::renderErrorPage(
                $e->statusCode(),
                self::titleForStatus($e->statusCode()),
                $e->getMessage(),
                $e
            );
            return;
        }

        /**
         * Если Router кидает RuntimeException с ROUTE_NOT_FOUND,
         * показываем нормальную 404.
         */
        if ($e instanceof RuntimeException && str_contains($e->getMessage(), 'ROUTE_NOT_FOUND')) {
            self::renderErrorPage(
                404,
                'Страница не найдена',
                'Маршрут для этой страницы не найден.',
                $e
            );
            return;
        }

        self::renderErrorPage(
            500,
            'Ошибка приложения',
            self::debug()
                ? $e->getMessage()
                : 'Произошла внутренняя ошибка. Попробуйте позже.',
            $e
        );
    }

    private static function renderValidationException(ValidationException $e): void
    {
        $errors = [];

        if (method_exists($e, 'errors')) {
            $errors = $e->errors();
        } elseif (method_exists($e, 'errorList')) {
            $errors = $e->errorList();
        }

        if (self::wantsJson()) {
            Response::json([
                'ok' => false,
                'error' => 'VALIDATION_ERROR',
                'message' => 'Проверьте заполнение формы.',
                'errors' => $errors,
            ], 422)->send();

            return;
        }

        $old = $_POST ?? [];

        unset($old['sessid'], $old['csrf_token'], $old['_method']);

        if (method_exists(Flash::class, 'old')) {
            Flash::old($old);
        }

        if (method_exists(Flash::class, 'error')) {
            Flash::error('Проверьте заполнение формы.');
        }

        redirect()->back()->send();
    }

    private static function renderErrorPage(
        int $status,
        string $title,
        string $message,
        ?Throwable $exception = null
    ): void {
        if (self::wantsJson()) {
            $payload = [
                'ok' => false,
                'error' => strtoupper(str_replace(' ', '_', $title)),
                'status' => $status,
                'message' => $message,
            ];

            if (self::debug() && $exception) {
                $payload['debug'] = [
                    'exception' => get_class($exception),
                    'file' => $exception->getFile(),
                    'line' => $exception->getLine(),
                    'trace' => explode("\n", $exception->getTraceAsString()),
                ];
            }

            Response::json($payload, $status)->send();
            return;
        }

        $html = self::renderErrorView($status, $title, $message, $exception);

        Response::html($html, $status)->send();
    }

    private static function renderErrorView(
        int $status,
        string $title,
        string $message,
        ?Throwable $exception = null
    ): string {
        $data = [
            'status' => $status,
            'title' => $title,
            'message' => $message,
            'debug' => self::debug(),
            'exception' => $exception,
        ];

        try {
            if (class_exists(View::class)) {
                $statusView = 'errors.' . $status;

                if (View::exists($statusView)) {
                    return View::render($statusView, $data);
                }

                if (View::exists('errors.error')) {
                    return View::render('errors.error', $data);
                }
            }
        } catch (Throwable) {
            // Если даже view ошибок сломался — отдаём fallback ниже.
        }

        return self::fallbackHtml($status, $title, $message, $exception);
    }

    private static function fallbackHtml(
        int $status,
        string $title,
        string $message,
        ?Throwable $exception = null
    ): string {
        $statusHtml = htmlspecialchars((string)$status, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
        $titleHtml = htmlspecialchars($title, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
        $messageHtml = htmlspecialchars($message, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');

        $debugHtml = '';

        if (self::debug() && $exception) {
            $debugHtml = '<div style="margin-top:20px;padding:16px;background:#111827;color:#fff;border-radius:12px;overflow:auto;">'
                . '<div><b>Exception:</b> ' . htmlspecialchars(get_class($exception), ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') . '</div>'
                . '<div><b>File:</b> ' . htmlspecialchars($exception->getFile(), ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') . '</div>'
                . '<div><b>Line:</b> ' . htmlspecialchars((string)$exception->getLine(), ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') . '</div>'
                . '<pre style="white-space:pre-wrap;">' . htmlspecialchars($exception->getTraceAsString(), ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') . '</pre>'
                . '</div>';
        }

        return <<<HTML
<!doctype html>
<html lang="ru">
<head>
    <meta charset="utf-8">
    <title>{$statusHtml} — {$titleHtml}</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">
</head>
<body style="margin:0;font-family:Arial,sans-serif;background:#f3f4f6;color:#111827;">
    <main style="max-width:780px;margin:80px auto;padding:24px;">
        <div style="background:#fff;border:1px solid #e5e7eb;border-radius:18px;padding:28px;box-shadow:0 10px 30px rgba(15,23,42,.08);">
            <div style="font-size:14px;color:#6b7280;margin-bottom:8px;">Ошибка {$statusHtml}</div>
            <h1 style="margin:0 0 12px;font-size:32px;">{$titleHtml}</h1>
            <p style="font-size:16px;line-height:1.6;margin:0;color:#374151;">{$messageHtml}</p>
            {$debugHtml}
        </div>
    </main>
</body>
</html>
HTML;
    }

    private static function wantsJson(): bool
    {
        $accept = (string)($_SERVER['HTTP_ACCEPT'] ?? '');
        $requestedWith = (string)($_SERVER['HTTP_X_REQUESTED_WITH'] ?? '');

        return str_contains($accept, 'application/json')
            || strtolower($requestedWith) === 'xmlhttprequest';
    }

    private static function debug(): bool
    {
        try {
            return (bool)Config::get('debug', false);
        } catch (Throwable) {
            return false;
        }
    }

    private static function titleForStatus(int $status): string
    {
        return match ($status) {
            400 => 'Некорректный запрос',
            401 => 'Необходима авторизация',
            403 => 'Доступ запрещён',
            404 => 'Страница не найдена',
            419 => 'Сессия истекла',
            422 => 'Ошибка валидации',
            default => 'Ошибка приложения',
        };
    }

    private static function logThrowable(Throwable $e): void
    {
        $message = sprintf(
            '[%s] %s in %s:%s',
            get_class($e),
            $e->getMessage(),
            $e->getFile(),
            $e->getLine()
        );

        error_log($message);
    }
}


---

Шаг 3. Добавь helpers abort(), abort_if(), abort_unless()

Открой:

/srv/bx/docroot/local/mvc/helpers.php

В конец файла добавь:

if (!function_exists('abort')) {
    /**
     * Laravel-like abort().
     *
     * Пример:
     * abort(404);
     * abort(403, 'Нет доступа');
     */
    function abort(int $status = 404, string $message = ''): void
    {
        throw new \Local\Mvc\Core\HttpException($status, $message);
    }
}

if (!function_exists('abort_if')) {
    /**
     * Laravel-like abort_if().
     *
     * Пример:
     * abort_if(!$user, 404);
     */
    function abort_if(bool $condition, int $status = 404, string $message = ''): void
    {
        if ($condition) {
            abort($status, $message);
        }
    }
}

if (!function_exists('abort_unless')) {
    /**
     * Laravel-like abort_unless().
     *
     * Пример:
     * abort_unless(Auth::check(), 403);
     */
    function abort_unless(bool $condition, int $status = 404, string $message = ''): void
    {
        if (!$condition) {
            abort($status, $message);
        }
    }
}

Теперь можно писать:

abort(404);

abort_if(!$note, 404);

abort_unless($canEdit, 403);


---

Шаг 4. Создай view ошибки

Создай папку:

/srv/bx/docroot/local/mvc_demo/Views/errors/

Создай файл:

/srv/bx/docroot/local/mvc_demo/Views/errors/error.php

Код:

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

$status = (int)($status ?? 500);
$title = (string)($title ?? 'Ошибка приложения');
$message = (string)($message ?? 'Произошла ошибка.');
$debug = (bool)($debug ?? false);
$exception = $exception ?? null;

?>

<div class="mvc-card">
    <div style="display:inline-flex;align-items:center;gap:8px;padding:6px 10px;border-radius:999px;background:#fef2f2;color:#991b1b;font-weight:600;margin-bottom:14px;">
        Ошибка <?= e($status) ?>
    </div>

    <h1 class="mvc-page-title">
        <?= e($title) ?>
    </h1>

    <p class="mvc-page-text">
        <?= e($message) ?>
    </p>

    <div class="mvc-info">
        <a href="<?= e(defined('LOCAL_MVC_PROJECT_URL') ? LOCAL_MVC_PROJECT_URL : '/') ?>">
            ← Вернуться на главную
        </a>
    </div>

    <?php if ($debug && $exception instanceof Throwable): ?>
        <div class="mvc-info" style="background:#111827;color:#fff;border-color:#111827;">
            <div style="margin-bottom:8px;">
                <b>Debug включён.</b> В production нужно поставить
                <span style="font-family:monospace;">debug => false</span>.
            </div>

            <div style="margin-bottom:6px;">
                <b>Exception:</b> <?= e(get_class($exception)) ?>
            </div>

            <div style="margin-bottom:6px;">
                <b>Message:</b> <?= e($exception->getMessage()) ?>
            </div>

            <div style="margin-bottom:6px;">
                <b>File:</b> <?= e($exception->getFile()) ?>
            </div>

            <div style="margin-bottom:12px;">
                <b>Line:</b> <?= e($exception->getLine()) ?>
            </div>

            <pre style="white-space:pre-wrap;overflow:auto;margin:0;font-size:12px;line-height:1.5;"><?= e($exception->getTraceAsString()) ?></pre>
        </div>
    <?php endif; ?>
</div>


---

Шаг 5. Для production выключи debug

В demo-проекте файл:

/srv/bx/docroot/local/mvc_demo/config.php

Сейчас на stage можно оставить:

'debug' => true,

Но для production должно быть:

'debug' => false,

То есть:

return [
    'app' => [
        'name' => 'MVC Demo',
    ],

    'debug' => false,

    // ...
];

На время разработки можешь держать true, но перед боевым использованием обязательно false.


---

Шаг 6. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Проверь несуществующую страницу:

/local/mvc_demo/test-not-found

Проверь несуществующую модель:

/local/mvc_demo/notes/999999/edit

Проверь вручную abort() в любом тестовом controller method:

abort(403, 'Тестовая проверка доступа');

При debug=true ты должен видеть подробности.

При debug=false пользователь должен видеть только красивую страницу ошибки без trace и путей файлов.


---

Что мы сейчас сделали:

Было: техническая ошибка прямо пользователю.
Стало: production-safe error handling.

Это один из самых важных шагов для production-фреймворка.