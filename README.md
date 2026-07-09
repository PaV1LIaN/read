Идём дальше. Следующий production-шаг — нормальные логи.

Сейчас ошибки могут уходить просто в error_log(). Для production лучше иметь свои файлы:

/local/mvc_demo/storage/logs/app-2026-07-09.log

И писать туда:

Log::info('Заметка создана', ['note_id' => 5]);
Log::error('Ошибка БД', ['exception' => $e]);


---

Шаг 1. Создай папки

Путь:

/srv/bx/docroot/local/mvc_demo/storage/logs

Если через терминал:

mkdir -p /srv/bx/docroot/local/mvc_demo/storage/logs
chown -R bitrix:bitrix /srv/bx/docroot/local/mvc_demo/storage
chmod -R 775 /srv/bx/docroot/local/mvc_demo/storage

Если пользователь веб-сервера другой — ставь владельца под него.


---

Шаг 2. Замени /local/mvc/Core/LogManager.php

Путь:

/srv/bx/docroot/local/mvc/Core/LogManager.php

Полный файл:

<?php

namespace Local\Mvc\Core;

use Throwable;

class LogManager
{
    private const LEVELS = [
        'debug' => 100,
        'info' => 200,
        'notice' => 250,
        'warning' => 300,
        'error' => 400,
        'critical' => 500,
        'alert' => 550,
        'emergency' => 600,
    ];

    public function debug(string $message, array $context = []): void
    {
        $this->log('debug', $message, $context);
    }

    public function info(string $message, array $context = []): void
    {
        $this->log('info', $message, $context);
    }

    public function notice(string $message, array $context = []): void
    {
        $this->log('notice', $message, $context);
    }

    public function warning(string $message, array $context = []): void
    {
        $this->log('warning', $message, $context);
    }

    public function error(string $message, array $context = []): void
    {
        $this->log('error', $message, $context);
    }

    public function critical(string $message, array $context = []): void
    {
        $this->log('critical', $message, $context);
    }

    public function alert(string $message, array $context = []): void
    {
        $this->log('alert', $message, $context);
    }

    public function emergency(string $message, array $context = []): void
    {
        $this->log('emergency', $message, $context);
    }

    public function exception(Throwable $e, string $message = 'Unhandled exception', array $context = []): void
    {
        $context['exception'] = [
            'class' => get_class($e),
            'message' => $e->getMessage(),
            'file' => $e->getFile(),
            'line' => $e->getLine(),
            'trace' => $e->getTraceAsString(),
        ];

        $this->error($message, $context);
    }

    public function log(string $level, string $message, array $context = []): void
    {
        $level = strtolower($level);

        if (!isset(self::LEVELS[$level])) {
            $level = 'info';
        }

        if (!$this->shouldLog($level)) {
            return;
        }

        $line = $this->formatLine($level, $message, $context);

        $path = $this->path();

        $dir = dirname($path);

        if (!is_dir($dir)) {
            @mkdir($dir, 0775, true);
        }

        if (!is_writable($dir)) {
            error_log($line);
            return;
        }

        @file_put_contents($path, $line . PHP_EOL, FILE_APPEND | LOCK_EX);
    }

    private function shouldLog(string $level): bool
    {
        $configuredLevel = strtolower((string)Config::get('logging.level', 'debug'));

        if (!isset(self::LEVELS[$configuredLevel])) {
            $configuredLevel = 'debug';
        }

        return self::LEVELS[$level] >= self::LEVELS[$configuredLevel];
    }

    private function path(): string
    {
        $path = (string)Config::get('logging.path', '');

        if ($path !== '') {
            return $this->replaceDate($path);
        }

        $root = defined('LOCAL_MVC_PROJECT_ROOT')
            ? rtrim(LOCAL_MVC_PROJECT_ROOT, '/')
            : rtrim($_SERVER['DOCUMENT_ROOT'] ?? sys_get_temp_dir(), '/');

        return $root . '/storage/logs/app-' . date('Y-m-d') . '.log';
    }

    private function replaceDate(string $path): string
    {
        return str_replace(
            ['{date}', '{Y-m-d}'],
            [date('Y-m-d'), date('Y-m-d')],
            $path
        );
    }

    private function formatLine(string $level, string $message, array $context): string
    {
        $context = $this->sanitizeContext($context);

        $record = [
            'time' => date('Y-m-d H:i:s'),
            'level' => strtoupper($level),
            'message' => $message,
            'url' => $this->currentUrl(),
            'method' => $_SERVER['REQUEST_METHOD'] ?? '',
            'user_id' => $this->userId(),
            'ip' => $_SERVER['REMOTE_ADDR'] ?? '',
            'context' => $context,
        ];

        return json_encode($record, JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES);
    }

    private function currentUrl(): string
    {
        $uri = (string)($_SERVER['REQUEST_URI'] ?? '');

        if ($uri === '') {
            return '';
        }

        $host = (string)($_SERVER['HTTP_HOST'] ?? '');

        return $host !== '' ? $host . $uri : $uri;
    }

    private function userId(): ?int
    {
        try {
            if (class_exists(Auth::class) && Auth::check()) {
                return Auth::id();
            }
        } catch (Throwable) {
            return null;
        }

        return null;
    }

    private function sanitizeContext(array $context): array
    {
        $hiddenKeys = [
            'password',
            'pass',
            'token',
            'access_token',
            'refresh_token',
            'authorization',
            'cookie',
            'sessid',
            'csrf',
            'csrf_token',
        ];

        foreach ($context as $key => $value) {
            $lowerKey = strtolower((string)$key);

            foreach ($hiddenKeys as $hiddenKey) {
                if (str_contains($lowerKey, $hiddenKey)) {
                    $context[$key] = '[hidden]';
                    continue 2;
                }
            }

            if (is_array($value)) {
                $context[$key] = $this->sanitizeContext($value);
            }
        }

        return $context;
    }
}


---

Шаг 3. Замени /local/mvc/Core/Logger.php

Путь:

/srv/bx/docroot/local/mvc/Core/Logger.php

Полный файл:

<?php

namespace Local\Mvc\Core;

use Throwable;

class Logger
{
    public static function debug(string $message, array $context = []): void
    {
        self::manager()->debug($message, $context);
    }

    public static function info(string $message, array $context = []): void
    {
        self::manager()->info($message, $context);
    }

    public static function notice(string $message, array $context = []): void
    {
        self::manager()->notice($message, $context);
    }

    public static function warning(string $message, array $context = []): void
    {
        self::manager()->warning($message, $context);
    }

    public static function error(string $message, array $context = []): void
    {
        self::manager()->error($message, $context);
    }

    public static function critical(string $message, array $context = []): void
    {
        self::manager()->critical($message, $context);
    }

    public static function exception(Throwable $e, string $message = 'Unhandled exception', array $context = []): void
    {
        self::manager()->exception($e, $message, $context);
    }

    private static function manager(): LogManager
    {
        try {
            return App::make(LogManager::class);
        } catch (Throwable) {
            return new LogManager();
        }
    }
}


---

Шаг 4. Проверь facade Log

Файл:

/srv/bx/docroot/local/mvc/Support/Facades/Log.php

Должен быть такой:

<?php

namespace Local\Mvc\Support\Facades;

use Local\Mvc\Core\LogManager;

class Log extends Facade
{
    protected static function accessor(): string
    {
        return LogManager::class;
    }
}


---

Шаг 5. Обнови config.php

Путь:

/srv/bx/docroot/local/mvc_demo/config.php

Добавь блок:

'logging' => [
    'level' => env('LOG_LEVEL', 'debug'),
    'path' => env(
        'LOG_PATH',
        rtrim(LOCAL_MVC_PROJECT_ROOT, '/') . '/storage/logs/app-{date}.log'
    ),
],

Например:

return [
    'app' => [
        'name' => env('APP_NAME', 'MVC Demo'),
        'description' => 'Тестовый проект на общем MVC-фреймворке',
    ],

    'debug' => env('APP_DEBUG', false),

    'logging' => [
        'level' => env('LOG_LEVEL', 'debug'),
        'path' => env(
            'LOG_PATH',
            rtrim(LOCAL_MVC_PROJECT_ROOT, '/') . '/storage/logs/app-{date}.log'
        ),
    ],

    // остальные блоки ниже...
];


---

Шаг 6. Обнови .env

Путь:

/srv/bx/docroot/local/mvc_demo/.env

Добавь:

LOG_LEVEL=debug

Для production потом лучше:

LOG_LEVEL=warning

То есть на production будут писаться только:

warning
error
critical
alert
emergency


---

Шаг 7. Обнови ErrorHandler.php

В файле:

/srv/bx/docroot/local/mvc/Core/ErrorHandler.php

Найди метод:

private static function logThrowable(Throwable $e): void

Замени его на:

private static function logThrowable(Throwable $e): void
{
    try {
        App::make(LogManager::class)->exception($e, 'Application exception');
        return;
    } catch (Throwable) {
        // fallback ниже
    }

    $message = sprintf(
        '[%s] %s in %s:%s',
        get_class($e),
        $e->getMessage(),
        $e->getFile(),
        $e->getLine()
    );

    error_log($message);
}


---

Шаг 8. Защити storage от браузера

Так как storage внутри docroot, его обязательно закрыть.

Для Nginx / Angie добавь:

location ^~ /local/mvc_demo/storage/ {
    deny all;
}

Проверка:

nginx -t
systemctl reload nginx

или для Angie:

angie -t
systemctl reload angie


---

Шаг 9. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Временно в любом контроллере можно добавить:

\Local\Mvc\Support\Facades\Log::info('Проверка логов', [
    'test' => true,
    'password' => '123456',
]);

Открой страницу, потом проверь файл:

/srv/bx/docroot/local/mvc_demo/storage/logs/app-2026-07-09.log

Внутри должно быть примерно так:

{"time":"2026-07-09 12:00:00","level":"INFO","message":"Проверка логов","url":"bitrix24-stage.gaz.ru/local/mvc_demo/notes","method":"GET","user_id":1,"ip":"...","context":{"test":true,"password":"[hidden]"}}


---

Что сделали:

Было: ошибки просто где-то в error_log.
Стало: свой production-лог проекта, с уровнями, датами и защитой секретов.

Это важная часть production-фреймворка: теперь ошибки можно нормально расследовать.