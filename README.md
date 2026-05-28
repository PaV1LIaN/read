Идём дальше. Сейчас сделаем middleware с параметрами.

Сейчас middleware у нас простые:

['auth']
['admin']
['csrf']

А в реальных проектах часто нужно так:

['group:1']

Это значит:

Пустить только пользователя из группы ID 1.

Или так:

['groups:1,7,12']

Это значит:

Пустить пользователя, если он есть хотя бы в одной из этих групп.

Это пригодится для проектов:

sitebuilder — доступ по группам
glab — админ/менеджер/ассистент
qr_opros — доступ к отчётам


---

1. Замени /local/mvc/Core/Middleware.php

<?php

namespace Local\Mvc\Core;

/**
 * Middleware
 *
 * Проверки, которые выполняются ДО контроллера.
 */
class Middleware
{
    public static function handle(array $middlewares, Request $request): ?Response
    {
        foreach ($middlewares as $middleware) {
            $middleware = trim((string)$middleware);

            if ($middleware === '') {
                continue;
            }

            $response = self::handleOne($middleware, $request);

            if ($response instanceof Response) {
                return $response;
            }
        }

        return null;
    }

    private static function handleOne(string $middleware, Request $request): ?Response
    {
        [$name, $argument] = self::parse($middleware);

        if ($name === 'auth') {
            if (Auth::check()) {
                return null;
            }

            return Response::json([
                'ok' => false,
                'error' => 'AUTH_REQUIRED',
                'details' => [
                    'message' => 'Нужно авторизоваться',
                ],
            ], 401);
        }

        if ($name === 'admin') {
            if (Auth::isAdmin()) {
                return null;
            }

            return Response::json([
                'ok' => false,
                'error' => 'ADMIN_REQUIRED',
                'details' => [
                    'message' => 'Нужны права администратора',
                ],
            ], 403);
        }

        /**
         * group:1
         *
         * Пускает только пользователя из одной конкретной группы.
         */
        if ($name === 'group') {
            $groupId = (int)$argument;

            if ($groupId > 0 && Auth::inGroup($groupId)) {
                return null;
            }

            return Response::json([
                'ok' => false,
                'error' => 'GROUP_REQUIRED',
                'details' => [
                    'message' => 'Недостаточно прав. Требуется группа: ' . $groupId,
                    'required_group' => $groupId,
                    'user_groups' => Auth::groups(),
                ],
            ], 403);
        }

        /**
         * groups:1,7,12
         *
         * Пускает пользователя, если он входит хотя бы в одну группу из списка.
         */
        if ($name === 'groups') {
            $requiredGroups = self::parseGroupList($argument);
            $userGroups = Auth::groups();

            foreach ($requiredGroups as $groupId) {
                if (in_array($groupId, $userGroups, true)) {
                    return null;
                }
            }

            return Response::json([
                'ok' => false,
                'error' => 'GROUPS_REQUIRED',
                'details' => [
                    'message' => 'Недостаточно прав. Требуется одна из групп.',
                    'required_groups' => $requiredGroups,
                    'user_groups' => $userGroups,
                ],
            ], 403);
        }

        if ($name === 'csrf') {
            if ($request->method() !== 'POST') {
                return null;
            }

            if (function_exists('check_bitrix_sessid') && check_bitrix_sessid()) {
                return null;
            }

            $headerSessid = (string)$request->header('X-Bitrix-Sessid', '');

            if (
                $headerSessid !== ''
                && function_exists('bitrix_sessid')
                && hash_equals((string)bitrix_sessid(), $headerSessid)
            ) {
                return null;
            }

            return Response::json([
                'ok' => false,
                'error' => 'BAD_SESSID',
                'details' => [
                    'message' => 'Неверный sessid. Обновите страницу и попробуйте снова.',
                ],
            ], 403);
        }

        return Response::json([
            'ok' => false,
            'error' => 'UNKNOWN_MIDDLEWARE',
            'details' => [
                'middleware' => $middleware,
            ],
        ], 500);
    }

    /**
     * Разобрать middleware.
     *
     * Было:
     * group:1
     *
     * Стало:
     * name = group
     * argument = 1
     */
    private static function parse(string $middleware): array
    {
        $parts = explode(':', $middleware, 2);

        $name = trim((string)($parts[0] ?? ''));
        $argument = trim((string)($parts[1] ?? ''));

        return [$name, $argument];
    }

    /**
     * Превратить строку "1,7,12" в массив [1, 7, 12].
     */
    private static function parseGroupList(string $argument): array
    {
        $items = explode(',', $argument);
        $groups = [];

        foreach ($items as $item) {
            $groupId = (int)trim($item);

            if ($groupId > 0) {
                $groups[] = $groupId;
            }
        }

        return array_values(array_unique($groups));
    }
}


---

2. Добавь тестовый метод в HomeController

Файл:

/local/mvc_demo/Controllers/HomeController.php

Добавь внутрь класса:

public function groupTest(): Response
{
    return $this->success([
        'message' => 'Доступ по группе разрешён.',
        'user_id' => \Local\Mvc\Core\Auth::id(),
        'groups' => \Local\Mvc\Core\Auth::groups(),
    ]);
}


---

3. Добавь маршрут в /local/mvc_demo/routes.php

Добавь публично рядом с остальными тестовыми маршрутами:

$router->get('/group-test', [HomeController::class, 'groupTest'], ['auth', 'group:1']);

Пример:

$router->get('/', [HomeController::class, 'index']);
$router->get('/about', [HomeController::class, 'about']);
$router->get('/ping', [HomeController::class, 'ping']);
$router->get('/group-test', [HomeController::class, 'groupTest'], ['auth', 'group:1']);

group:1 — это обычно группа администраторов Битрикса. Если у вас другая группа, поменяешь ID.


---

4. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/group-test

Если текущий пользователь в группе 1, будет:

{
  "ok": true,
  "data": {
    "message": "Доступ по группе разрешён.",
    "user_id": 1,
    "groups": [1, 2]
  }
}

Если не в группе, будет:

{
  "ok": false,
  "error": "GROUP_REQUIRED",
  "details": {
    "message": "Недостаточно прав. Требуется группа: 1",
    "required_group": 1,
    "user_groups": [...]
  }
}


---

Что мы сделали

Теперь middleware могут быть не только простыми:

['auth']
['admin']
['csrf']

но и с параметрами:

['group:1']
['groups:1,7,12']

Это уже очень полезно для реальных проектов.

Например:

$router->group(['middleware' => ['auth', 'groups:10,11']], function (Router $router) {
    $router->get('/reports', [ReportController::class, 'index']);
});

Простыми словами:

Пустить в отчёты только пользователей из групп 10 или 11.

Следующий шаг — сделать middleware для ролей проекта, чтобы можно было не только по группам Битрикса, а по своим ролям: owner, admin, editor, viewer.