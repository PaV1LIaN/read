Идём дальше. После нормальных страниц ошибок следующий production-шаг — .env / environment config.

Сейчас в config.php у нас может быть так:

'debug' => true,

Для production это опасно: можно забыть выключить debug и показать пользователю пути файлов, trace, SQL-ошибки.

Сделаем как в Laravel:

'debug' => env('APP_DEBUG', false),


---

Шаг 1. Создай /local/mvc/Core/Env.php

Путь:

/srv/bx/docroot/local/mvc/Core/Env.php

Код:

<?php

namespace Local\Mvc\Core;

class Env
{
    private static array $values = [];

    private static bool $loaded = false;

    public static function load(string $path): void
    {
        if (self::$loaded) {
            return;
        }

        self::$loaded = true;

        if (!is_file($path)) {
            return;
        }

        $lines = file($path, FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES);

        if (!is_array($lines)) {
            return;
        }

        foreach ($lines as $line) {
            $line = trim($line);

            if ($line === '' || str_starts_with($line, '#')) {
                continue;
            }

            if (!str_contains($line, '=')) {
                continue;
            }

            [$key, $value] = explode('=', $line, 2);

            $key = trim($key);
            $value = trim($value);

            if ($key === '') {
                continue;
            }

            $value = self::normalizeValue($value);

            self::$values[$key] = $value;

            if (getenv($key) === false) {
                putenv($key . '=' . self::stringValue($value));
            }

            $_ENV[$key] = $value;
            $_SERVER[$key] = $value;
        }
    }

    public static function get(string $key, mixed $default = null): mixed
    {
        if (array_key_exists($key, self::$values)) {
            return self::$values[$key];
        }

        $value = $_ENV[$key] ?? $_SERVER[$key] ?? getenv($key);

        if ($value === false || $value === null) {
            return $default;
        }

        return self::normalizeValue((string)$value);
    }

    private static function normalizeValue(string $value): mixed
    {
        $value = trim($value);

        if (
            (str_starts_with($value, '"') && str_ends_with($value, '"'))
            || (str_starts_with($value, "'") && str_ends_with($value, "'"))
        ) {
            $value = substr($value, 1, -1);
        }

        $lower = strtolower($value);

        return match ($lower) {
            'true', '(true)' => true,
            'false', '(false)' => false,
            'null', '(null)' => null,
            'empty', '(empty)' => '',
            default => $value,
        };
    }

    private static function stringValue(mixed $value): string
    {
        if ($value === true) {
            return 'true';
        }

        if ($value === false) {
            return 'false';
        }

        if ($value === null) {
            return '';
        }

        return (string)$value;
    }
}


---

Шаг 2. Обнови /local/mvc/bootstrap.php

Путь:

/srv/bx/docroot/local/mvc/bootstrap.php

После регистрации autoload и подключения helpers добавь загрузку .env.

Идея такая: если demo-проект находится здесь:

/srv/bx/docroot/local/mvc_demo

то .env будет лежать здесь:

/srv/bx/docroot/local/mvc_demo/.env

Добавь в bootstrap.php такой блок:

if (defined('LOCAL_MVC_PROJECT_ROOT')) {
    \Local\Mvc\Core\Env::load(rtrim(LOCAL_MVC_PROJECT_ROOT, '/') . '/.env');
}

Важно: этот блок должен идти после autoload, потому что класс Env должен уже подгружаться.

Примерно так:

require_once __DIR__ . '/helpers.php';

if (defined('LOCAL_MVC_PROJECT_ROOT')) {
    \Local\Mvc\Core\Env::load(rtrim(LOCAL_MVC_PROJECT_ROOT, '/') . '/.env');
}


---

Шаг 3. Добавь helper env()

Открой:

/srv/bx/docroot/local/mvc/helpers.php

В конец файла добавь:

if (!function_exists('env')) {
    /**
     * Laravel-like env().
     *
     * Пример:
     * env('APP_DEBUG', false)
     */
    function env(string $key, mixed $default = null): mixed
    {
        return \Local\Mvc\Core\Env::get($key, $default);
    }
}


---

Шаг 4. Создай .env для demo-проекта

Путь:

/srv/bx/docroot/local/mvc_demo/.env

Код:

APP_ENV=local
APP_DEBUG=true
APP_NAME="MVC Demo"

DB_DEFAULT=projects
DB_PROJECTS_SCHEMA=mvc

LOG_LEVEL=debug

На stage / разработке можно оставить:

APP_DEBUG=true

На production должно быть:

APP_ENV=production
APP_DEBUG=false


---

Шаг 5. Обнови /local/mvc_demo/config.php

Путь:

/srv/bx/docroot/local/mvc_demo/config.php

Найди:

'debug' => true,

Замени на:

'debug' => env('APP_DEBUG', false),

Если есть app.name, можно сделать так:

'app' => [
    'name' => env('APP_NAME', 'MVC Demo'),
    'description' => 'Тестовый проект на общем MVC-фреймворке',
],

И в блоке database можно сделать schema через env:

'database' => [
    'default' => env('DB_DEFAULT', 'bitrix'),

    'connections' => [
        'bitrix' => [
            'driver' => 'bitrix',
        ],

        'projects' => [
            'driver' => 'pg_master',
            'schema' => env('DB_PROJECTS_SCHEMA', 'mvc'),
        ],
    ],

    'migrations' => [
        'connection' => 'projects',
        'table' => 'mvc.migrations',
    ],

    'seeders' => [
        \Local\MvcDemo\Database\Seeders\DatabaseSeeder::class,
    ],
],


---

Шаг 6. Защити .env от открытия через браузер

Так как .env лежит внутри docroot, его обязательно надо закрыть.

Для Nginx / Angie добавь правило в конфиг сайта:

location ~ /\. {
    deny all;
    access_log off;
    log_not_found off;
}

Минимально конкретно для .env:

location ~ /\.env {
    deny all;
    access_log off;
    log_not_found off;
}

После изменения конфига:

nginx -t
systemctl reload nginx

Если у вас Angie:

angie -t
systemctl reload angie


---

Шаг 7. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Проверь, что debug берётся из .env.

Временно поставь в:

/srv/bx/docroot/local/mvc_demo/.env

APP_DEBUG=false

Открой несуществующую страницу:

/local/mvc_demo/test-not-found

Trace и пути файлов не должны показываться.

Потом верни для разработки:

APP_DEBUG=true

И снова сбрось OPcache.


---

Что мы сделали:

Было: debug и настройки захардкожены в config.php.
Стало: настройки окружения вынесены в .env.

Это production-подход: один и тот же код можно держать на stage и production, а различия хранить в .env.