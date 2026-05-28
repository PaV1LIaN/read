Да, это из-за ?page=2.

Сейчас при переходе на:

/local/mvc_demo/admin/users?page=2

в route может попасть не просто:

/admin/users

а вот так:

/admin/users?page=2

И Router ищет маршрут:

/admin/users?page=2

А у нас есть только:

/admin/users

Поэтому 404.

Нужно поправить Request::path(), чтобы он отрезал ?page=2 от маршрута.


---

Замени метод path() в /local/mvc/Core/Request.php

Найди метод:

public function path(): string

и замени его полностью на этот:

public function path(): string
{
    /**
     * 1. Сначала проверяем старый вариант через ?route=
     *
     * Например:
     * /local/mvc_demo/?route=/admin/users
     *
     * Или после urlrewrite:
     * route=/admin/users?page=2
     */
    $routeFromGet = trim((string)$this->get('route', ''));

    if ($routeFromGet !== '') {
        /**
         * Важно:
         * если внутрь route случайно попал query string,
         * например /admin/users?page=2,
         * отрезаем всё после ?.
         */
        $routePath = parse_url($routeFromGet, PHP_URL_PATH);

        if (!is_string($routePath) || $routePath === '') {
            $routePath = $routeFromGet;
        }

        return $this->normalizePath($routePath);
    }

    /**
     * 2. Если route нет, берём настоящий адрес из REQUEST_URI.
     *
     * Например:
     * /local/mvc_demo/admin/users?page=2
     */
    $requestUri = (string)$this->server('REQUEST_URI', '/');

    /**
     * Убираем query string.
     *
     * Было:
     * /local/mvc_demo/admin/users?page=2
     *
     * Стало:
     * /local/mvc_demo/admin/users
     */
    $uriPath = parse_url($requestUri, PHP_URL_PATH);

    if (!is_string($uriPath) || $uriPath === '') {
        $uriPath = '/';
    }

    /**
     * SCRIPT_NAME обычно такой:
     * /local/mvc_demo/index.php
     *
     * Нам нужно получить базовую папку:
     * /local/mvc_demo
     */
    $scriptName = (string)$this->server('SCRIPT_NAME', '');
    $scriptDir = rtrim(str_replace('\\', '/', dirname($scriptName)), '/');

    /**
     * Убираем базовую папку проекта.
     *
     * Было:
     * /local/mvc_demo/admin/users
     *
     * Стало:
     * /admin/users
     */
    if ($scriptDir !== '' && $scriptDir !== '/' && str_starts_with($uriPath, $scriptDir)) {
        $uriPath = substr($uriPath, strlen($scriptDir));
    }

    /**
     * Если вдруг адрес вида:
     * /local/mvc_demo/index.php/admin/users
     *
     * убираем index.php.
     */
    if (str_starts_with($uriPath, '/index.php')) {
        $uriPath = substr($uriPath, strlen('/index.php'));
    }

    return $this->normalizePath($uriPath);
}


---

Потом сбрось OPcache

opcache_reset();
echo 'OPcache reset OK';


---

Проверь

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/admin/users

Потом нажми на страницу 2.

Должно открыться:

https://bitrix24-stage.gaz.ru/local/mvc_demo/admin/users?page=2

И Router должен увидеть маршрут:

/admin/users

А page=2 останется доступен здесь:

$this->request->get('page', 1)

То есть теперь:

route = /admin/users
page = 2

А не:

route = /admin/users?page=2