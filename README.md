Идём дальше. Следующий production-шаг — нормальная Session abstraction.

Сейчас у нас Flash, old(), ошибки форм и CSRF частично завязаны на сырые $_SESSION. Для production лучше сделать свой слой, как в Laravel:

session()->get('key');
session()->put('key', 'value');
session()->flash('success', 'Готово');
session()->pull('key');


---

Шаг 1. Создай /local/mvc/Core/SessionManager.php

Путь:

/srv/bx/docroot/local/mvc/Core/SessionManager.php

Код:

<?php

namespace Local\Mvc\Core;

class SessionManager
{
    public function __construct()
    {
        $this->start();
    }

    public function start(): void
    {
        if (session_status() === PHP_SESSION_NONE && !headers_sent()) {
            session_start();
        }

        if (!isset($_SESSION) || !is_array($_SESSION)) {
            $_SESSION = [];
        }
    }

    public function has(string $key): bool
    {
        $this->start();

        return array_key_exists($key, $_SESSION);
    }

    public function get(string $key, mixed $default = null): mixed
    {
        $this->start();

        return $_SESSION[$key] ?? $default;
    }

    public function put(string $key, mixed $value): void
    {
        $this->start();

        $_SESSION[$key] = $value;
    }

    public function forget(string $key): void
    {
        $this->start();

        unset($_SESSION[$key]);
    }

    public function pull(string $key, mixed $default = null): mixed
    {
        $this->start();

        $value = $_SESSION[$key] ?? $default;

        unset($_SESSION[$key]);

        return $value;
    }

    public function flash(string $key, mixed $value): void
    {
        $this->start();

        $_SESSION['_flash'][$key] = $value;
    }

    public function getFlash(string $key, mixed $default = null): mixed
    {
        $this->start();

        return $_SESSION['_flash'][$key] ?? $default;
    }

    public function pullFlash(string $key, mixed $default = null): mixed
    {
        $this->start();

        $value = $_SESSION['_flash'][$key] ?? $default;

        unset($_SESSION['_flash'][$key]);

        if (empty($_SESSION['_flash'])) {
            unset($_SESSION['_flash']);
        }

        return $value;
    }

    public function regenerate(): void
    {
        $this->start();

        if (!headers_sent()) {
            session_regenerate_id(true);
        }
    }

    public function all(): array
    {
        $this->start();

        return $_SESSION;
    }
}


---

Шаг 2. Зарегистрируй SessionManager в App.php

Путь:

/srv/bx/docroot/local/mvc/Core/App.php

В методе run() найди место, где регистрируются singleton:

$container->singleton(\Local\Mvc\Core\LogManager::class, \Local\Mvc\Core\LogManager::class);
$container->singleton(\Local\Mvc\Core\ConfigManager::class, \Local\Mvc\Core\ConfigManager::class);

Добавь рядом:

$container->singleton(\Local\Mvc\Core\SessionManager::class, \Local\Mvc\Core\SessionManager::class);


---

Шаг 3. Добавь helper session()

Путь:

/srv/bx/docroot/local/mvc/helpers.php

В конец файла добавь:

if (!function_exists('session')) {
    /**
     * Laravel-like session().
     *
     * Примеры:
     * session()->get('key')
     * session()->put('key', 'value')
     * session('key', 'default')
     */
    function session(?string $key = null, mixed $default = null): mixed
    {
        $manager = \Local\Mvc\Core\App::make(\Local\Mvc\Core\SessionManager::class);

        if ($key === null) {
            return $manager;
        }

        return $manager->get($key, $default);
    }
}

Теперь можно писать:

session()->put('test', 123);

$value = session('test');


---

Шаг 4. Обнови /local/mvc/Core/Flash.php

Полностью замени файл:

<?php

namespace Local\Mvc\Core;

class Flash
{
    private const KEY_MESSAGES = 'messages';

    private const KEY_OLD = 'old';

    public static function success(string $message): void
    {
        self::message('success', $message);
    }

    public static function error(string $message): void
    {
        self::message('error', $message);
    }

    public static function info(string $message): void
    {
        self::message('info', $message);
    }

    public static function message(string $type, string $message): void
    {
        $messages = self::session()->getFlash(self::KEY_MESSAGES, []);

        if (!is_array($messages)) {
            $messages = [];
        }

        $messages[] = [
            'type' => $type,
            'message' => $message,
        ];

        self::session()->flash(self::KEY_MESSAGES, $messages);
    }

    public static function all(): array
    {
        $messages = self::session()->pullFlash(self::KEY_MESSAGES, []);

        return is_array($messages) ? $messages : [];
    }

    public static function old(array $data): void
    {
        self::session()->flash(self::KEY_OLD, $data);
    }

    public static function getOld(): array
    {
        $old = self::session()->pullFlash(self::KEY_OLD, []);

        return is_array($old) ? $old : [];
    }

    private static function session(): SessionManager
    {
        return App::make(SessionManager::class);
    }
}


---

Шаг 5. Проверь Controller.php

Путь:

/srv/bx/docroot/local/mvc/Core/Controller.php

В render() должно быть что-то похожее:

$flash = Flash::all();
$old = Flash::getOld();

ViewData::set('old', $old);

Если у тебя там уже это есть — ничего менять не надо.

Главное, чтобы во view передавались:

'flash' => $flash,
'old' => $old,


---

Шаг 6. Добавь facade Session

Создай файл:

/srv/bx/docroot/local/mvc/Support/Facades/Session.php

Код:

<?php

namespace Local\Mvc\Support\Facades;

use Local\Mvc\Core\SessionManager;

class Session extends Facade
{
    protected static function accessor(): string
    {
        return SessionManager::class;
    }
}

Теперь можно писать:

use Local\Mvc\Support\Facades\Session;

Session::put('key', 'value');
$value = Session::get('key');


---

Шаг 7. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Для проверки можешь временно в любом методе контроллера написать:

session()->put('mvc_test', 'Работает');

Flash::success('Session работает: ' . session('mvc_test'));

return redirect()->route('notes.index');

После перехода на /notes должен появиться flash-message.


---

Что мы сделали

Было:

$_SESSION['_flash']
$_SESSION['old']

Стало Laravel-like:

session()->put('key', 'value');
session()->get('key');
session()->flash('success', 'Готово');
session()->pull('key');

Главная мысль:

Теперь ядро фреймворка не зависит напрямую от $_SESSION.
У нас появился нормальный слой SessionManager.

Это важный production-шаг, потому что дальше через него можно делать авторизацию, flash, remember-поля, CSRF и системные уведомления аккуратно.