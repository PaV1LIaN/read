Идём дальше. Сейчас сделаем Logger — журнал событий.

Зачем нужен Logger

Когда что-то ломается, сейчас ошибка уходит в стандартный error_log.

Но лучше, чтобы каждый проект мог писать свои логи сюда:

/local/mvc_demo/logs/app.log

Простыми словами:

Logger — это тетрадка, куда приложение записывает:
- ошибки
- важные действия
- отладочную информацию

Например:

Logger::info('Пользователь открыл форму');
Logger::error('Ошибка сохранения сайта');


---

1. Создай /local/mvc/Core/Logger.php

<?php

namespace Local\Mvc\Core;

/**
 * Logger
 *
 * Простой логгер MVC-фреймворка.
 *
 * Он пишет сообщения в файл:
 * /local/проект/logs/app.log
 */
class Logger
{
    public static function info(string $message, array $context = []): void
    {
        self::write('INFO', $message, $context);
    }

    public static function warning(string $message, array $context = []): void
    {
        self::write('WARNING', $message, $context);
    }

    public static function error(string $message, array $context = []): void
    {
        self::write('ERROR', $message, $context);
    }

    public static function debug(string $message, array $context = []): void
    {
        if (!defined('LOCAL_MVC_DEBUG') || LOCAL_MVC_DEBUG !== true) {
            return;
        }

        self::write('DEBUG', $message, $context);
    }

    private static function write(string $level, string $message, array $context = []): void
    {
        $logFile = self::logFile();

        $dir = dirname($logFile);

        if (!is_dir($dir)) {
            @mkdir($dir, 0775, true);
        }

        $line = self::formatLine($level, $message, $context);

        @file_put_contents($logFile, $line, FILE_APPEND | LOCK_EX);
    }

    private static function formatLine(string $level, string $message, array $context = []): string
    {
        $date = date('Y-m-d H:i:s');

        $contextText = '';

        if (!empty($context)) {
            $contextText = ' ' . json_encode($context, JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES);
        }

        return '[' . $date . '] [' . $level . '] ' . $message . $contextText . PHP_EOL;
    }

    private static function logFile(): string
    {
        if (defined('LOCAL_MVC_LOG_FILE')) {
            return (string)LOCAL_MVC_LOG_FILE;
        }

        if (defined('LOCAL_MVC_PROJECT_ROOT')) {
            return rtrim((string)LOCAL_MVC_PROJECT_ROOT, '/') . '/logs/app.log';
        }

        return $_SERVER['DOCUMENT_ROOT'] . '/local/mvc/logs/app.log';
    }
}


---

2. Обнови /local/mvc/Core/ErrorHandler.php

Внизу файла найди метод:

private static function log(string $message, string $file, int $line): void
{
    error_log('[LOCAL_MVC_ERROR] ' . $message . ' in ' . $file . ':' . $line);
}

Замени его на:

private static function log(string $message, string $file, int $line): void
{
    Logger::error($message, [
        'file' => $file,
        'line' => $line,
    ]);

    error_log('[LOCAL_MVC_ERROR] ' . $message . ' in ' . $file . ':' . $line);
}

Теперь ошибки будут писаться и в системный лог, и в файл проекта.


---

3. Создай папку логов

Создай папку:

/local/mvc_demo/logs/

Права желательно такие, чтобы веб-сервер мог туда писать.

Можно через Linux:

mkdir -p /srv/bx/docroot/local/mvc_demo/logs
chmod 775 /srv/bx/docroot/local/mvc_demo/logs

Если пользователь веб-сервера другой, может понадобиться chown, но сначала проверь без этого.


---

4. Добавим тест логгера

Открой:

/local/mvc_demo/Controllers/HomeController.php

Добавь сверху:

use Local\Mvc\Core\Logger;

И внутрь класса добавь метод:

public function logTest(): Response
{
    Logger::info('Открыта тестовая страница логгера', [
        'user_id' => \Local\Mvc\Core\Auth::id(),
        'path' => $this->request->path(),
    ]);

    Logger::debug('Это debug-сообщение. Оно пишется только когда LOCAL_MVC_DEBUG = true');

    return $this->success([
        'message' => 'Лог записан',
        'file' => '/local/mvc_demo/logs/app.log',
    ]);
}


---

5. Обнови /local/mvc_demo/routes.php

Добавь маршрут:

$router->get('/log-test', [HomeController::class, 'logTest']);

Например рядом с публичными:

$router->get('/', [HomeController::class, 'index']);

$router->get('/about', [HomeController::class, 'about']);

$router->get('/ping', [HomeController::class, 'ping']);

$router->get('/log-test', [HomeController::class, 'logTest']);

$router->get('/form', [FormController::class, 'index']);


---

6. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/log-test

Должен быть JSON:

{
  "ok": true,
  "data": {
    "message": "Лог записан",
    "file": "/local/mvc_demo/logs/app.log"
  }
}

Потом проверь файл:

/srv/bx/docroot/local/mvc_demo/logs/app.log

Там должны появиться строки примерно такие:

[2026-05-28 10:15:00] [INFO] Открыта тестовая страница логгера {"user_id":1,"path":"/log-test"}
[2026-05-28 10:15:00] [DEBUG] Это debug-сообщение. Оно пишется только когда LOCAL_MVC_DEBUG = true


---

Что мы сделали

Теперь у фреймворка есть общий инструмент:

Logger::info(...)
Logger::warning(...)
Logger::error(...)
Logger::debug(...)

Он пригодится в реальных проектах.

Например в sitebuilder:

Logger::info('Создан сайт', [
    'site_id' => $siteId,
    'user_id' => Auth::id(),
]);

Или при ошибке:

Logger::error('Ошибка создания страницы', [
    'message' => $e->getMessage(),
    'user_id' => Auth::id(),
]);

Главная мысль:

ErrorHandler ловит ошибки.
Logger записывает их в файл.

Следующий шаг — сделать Config, чтобы настройки проекта лежали в одном месте: название проекта, debug, путь к логам, база, timezone и прочее.