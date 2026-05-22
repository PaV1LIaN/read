Отлично, идём дальше.

Сейчас сделаем:

/local/mvc/Core/Auth.php

Auth — это помощник для авторизации.

Простыми словами, он будет отвечать на вопросы:

Пользователь авторизован?
Какой у него ID?
Какой у него логин?
Как получить ФИО?
Как запретить доступ гостю?


---

1. Создай /local/mvc/Core/Auth.php

<?php

namespace Local\Mvc\Core;

use CUser;

/**
 * Auth
 *
 * Помощник для работы с текущим пользователем Битрикса.
 */
class Auth
{
    /**
     * Получить объект пользователя Битрикса.
     */
    public static function user(): ?CUser
    {
        global $USER;

        return $USER instanceof CUser ? $USER : null;
    }

    /**
     * Проверить, авторизован ли пользователь.
     */
    public static function check(): bool
    {
        $user = self::user();

        return $user !== null && $user->IsAuthorized();
    }

    /**
     * Получить ID текущего пользователя.
     */
    public static function id(): int
    {
        $user = self::user();

        if ($user === null || !$user->IsAuthorized()) {
            return 0;
        }

        return (int)$user->GetID();
    }

    /**
     * Получить логин пользователя.
     */
    public static function login(): string
    {
        $user = self::user();

        if ($user === null || !$user->IsAuthorized()) {
            return '';
        }

        return (string)$user->GetLogin();
    }

    /**
     * Получить email пользователя.
     */
    public static function email(): string
    {
        $userId = self::id();

        if ($userId <= 0) {
            return '';
        }

        $rs = \CUser::GetByID($userId);

        if ($row = $rs->Fetch()) {
            return (string)($row['EMAIL'] ?? '');
        }

        return '';
    }

    /**
     * Получить ФИО пользователя.
     */
    public static function name(): string
    {
        $userId = self::id();

        if ($userId <= 0) {
            return '';
        }

        $rs = \CUser::GetByID($userId);

        if ($row = $rs->Fetch()) {
            $lastName = trim((string)($row['LAST_NAME'] ?? ''));
            $name = trim((string)($row['NAME'] ?? ''));
            $secondName = trim((string)($row['SECOND_NAME'] ?? ''));

            $fullName = trim($lastName . ' ' . $name . ' ' . $secondName);

            if ($fullName !== '') {
                return $fullName;
            }

            return (string)($row['LOGIN'] ?? '');
        }

        return '';
    }

    /**
     * Проверить, является ли пользователь администратором.
     */
    public static function isAdmin(): bool
    {
        $user = self::user();

        return $user !== null && $user->IsAdmin();
    }

    /**
     * Получить группы пользователя.
     */
    public static function groups(): array
    {
        $user = self::user();

        if ($user === null || !$user->IsAuthorized()) {
            return [];
        }

        $groups = $user->GetUserGroupArray();

        return is_array($groups) ? array_map('intval', $groups) : [];
    }

    /**
     * Проверить, входит ли пользователь в группу.
     */
    public static function inGroup(int $groupId): bool
    {
        return in_array($groupId, self::groups(), true);
    }
}


---

2. Добавим защиту в базовый Controller

Теперь удобно сделать методы:

$this->requireAuth();
$this->requireAdmin();

Открой:

/local/mvc/Core/Controller.php

И перед последней закрывающей скобкой класса добавь методы:

/**
     * Запретить доступ гостям.
     */
    protected function requireAuth(): ?Response
    {
        if (Auth::check()) {
            return null;
        }

        return $this->error('AUTH_REQUIRED', [
            'message' => 'Нужно авторизоваться',
        ], 401);
    }

    /**
     * Запретить доступ всем, кроме администраторов.
     */
    protected function requireAdmin(): ?Response
    {
        if (Auth::isAdmin()) {
            return null;
        }

        return $this->error('ADMIN_REQUIRED', [
            'message' => 'Нужны права администратора',
        ], 403);
    }

То есть в контроллере можно будет писать:

if ($response = $this->requireAuth()) {
    return $response;
}

И если пользователь не авторизован — метод сразу вернёт ошибку.


---

3. Добавим тестовый метод в mvc_demo

Открой:

/local/mvc_demo/Controllers/HomeController.php

Добавь сверху use:

use Local\Mvc\Core\Auth;

И внутрь класса добавь метод:

public function me(): Response
    {
        if ($response = $this->requireAuth()) {
            return $response;
        }

        return $this->success([
            'id' => Auth::id(),
            'login' => Auth::login(),
            'name' => Auth::name(),
            'email' => Auth::email(),
            'is_admin' => Auth::isAdmin(),
            'groups' => Auth::groups(),
        ]);
    }

Полный контроллер может выглядеть так:

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\Auth;
use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Response;

class HomeController extends Controller
{
    public function index(): Response
    {
        $name = (string)$this->request->get('name', 'Гость');

        return $this->render('home/index', [
            'title' => 'MVC Demo',
            'message' => 'Привет, ' . $name . '! Это отдельный проект, который использует общий фреймворк.',
        ]);
    }

    public function about(): Response
    {
        return $this->render('home/index', [
            'title' => 'О проекте MVC Demo',
            'message' => 'Этот проект лежит в /local/mvc_demo, а фреймворк лежит отдельно в /local/mvc.',
        ]);
    }

    public function ping(): Response
    {
        return $this->success([
            'message' => 'pong',
            'project' => 'mvc_demo',
            'framework' => 'local_mvc',
            'path' => $this->request->path(),
        ]);
    }

    public function me(): Response
    {
        if ($response = $this->requireAuth()) {
            return $response;
        }

        return $this->success([
            'id' => Auth::id(),
            'login' => Auth::login(),
            'name' => Auth::name(),
            'email' => Auth::email(),
            'is_admin' => Auth::isAdmin(),
            'groups' => Auth::groups(),
        ]);
    }
}


---

4. Обнови /local/mvc_demo/routes.php

Добавь маршрут:

$router->get('/me', [HomeController::class, 'me']);

Полный файл:

<?php

use Local\Mvc\Core\Router;
use Local\MvcDemo\Controllers\HomeController;

/** @var Router $router */

$router->get('/', [HomeController::class, 'index']);

$router->get('/about', [HomeController::class, 'about']);

$router->get('/ping', [HomeController::class, 'ping']);

$router->get('/me', [HomeController::class, 'me']);


---

5. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/me

Если ты авторизован в Битриксе, должен быть JSON примерно такой:

{
  "ok": true,
  "data": {
    "id": 1,
    "login": "admin",
    "name": "Иван Иванов",
    "email": "admin@example.ru",
    "is_admin": true,
    "groups": [1, 2, 3]
  }
}

Если не авторизован:

{
  "ok": false,
  "error": "AUTH_REQUIRED",
  "details": {
    "message": "Нужно авторизоваться"
  }
}


---

Что мы сделали

Теперь фреймворк умеет работать с пользователем Битрикса.

Появился общий инструмент:

Auth::check()
Auth::id()
Auth::login()
Auth::name()
Auth::email()
Auth::isAdmin()
Auth::groups()
Auth::inGroup(1)

И в любом проекте можно писать:

if ($response = $this->requireAuth()) {
    return $response;
}

или:

if ($response = $this->requireAdmin()) {
    return $response;
}

Это уже очень полезно для будущих проектов:

sitebuilder
glab
qr_opros
админки
API
диск-компонента

Следующий шаг — сделать Middleware, чтобы не писать проверку авторизации вручную в каждом методе. Например, чтобы маршрут сразу говорил: “сюда пускать только авторизованных”.