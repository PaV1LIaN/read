Понял. Значит проблема не в Router, а в том, что запрос /about вообще не попадает в наш /local/mvc/index.php.

Битрикс перехватывает адрес и думает:

/about — это обычная страница Битрикса
страницы нет
значит показать 404 / карту сайта

Нам нужно, чтобы адрес был не просто:

/about

а именно:

/local/mvc/about

И чтобы веб-сервер или Битрикс отправлял его в:

/local/mvc/index.php


---

Сначала проверь правильно адрес

Открывать надо так:

https://bitrix24-stage.gaz.ru/local/mvc/about

А не так:

https://bitrix24-stage.gaz.ru/about

Если открываешь именно /about, Битрикс правильно кидает на свою 404/карту сайта, потому что это адрес в корне сайта.


---

Быстрое рабочее решение через ?route=

Пока красивые URL не настроены, используй:

https://bitrix24-stage.gaz.ru/local/mvc/?route=/about

Если так работает — значит MVC полностью живой.
Не работает только прокидывание красивого адреса.


---

Правильное решение для Битрикса: добавить правило в /urlrewrite.php

На Bitrix часто запросы идут через /urlrewrite.php.

Открой файл:

/srv/bx/docroot/urlrewrite.php

Там будет массив $arUrlRewrite.

В самое начало массива добавь правило:

[
    'CONDITION' => '#^/local/mvc(?:/(.*))?/?$#',
    'RULE' => 'MVC_ROUTE=$1',
    'ID' => '',
    'PATH' => '/local/mvc/index.php',
    'SORT' => 10,
],

Примерно должно получиться так:

<?php

$arUrlRewrite = [
    [
        'CONDITION' => '#^/local/mvc(?:/(.*))?/?$#',
        'RULE' => 'MVC_ROUTE=$1',
        'ID' => '',
        'PATH' => '/local/mvc/index.php',
        'SORT' => 10,
    ],

    // остальные правила Битрикса ниже
];


---

Потом поправим Request.php

Нужно, чтобы он понимал переменную MVC_ROUTE.

В файле:

/local/mvc/Core/Request.php

найди метод:

public function path(): string

И в самое начало метода добавь проверку:

$routeFromBitrixRewrite = trim((string)$this->get('MVC_ROUTE', ''));

if ($routeFromBitrixRewrite !== '') {
    return $this->normalizePath($routeFromBitrixRewrite);
}

То есть начало метода должно стать таким:

public function path(): string
{
    /**
     * Вариант через Bitrix urlrewrite.php:
     *
     * /local/mvc/about
     * превращается в:
     * /local/mvc/index.php?MVC_ROUTE=about
     */
    $routeFromBitrixRewrite = trim((string)$this->get('MVC_ROUTE', ''));

    if ($routeFromBitrixRewrite !== '') {
        return $this->normalizePath($routeFromBitrixRewrite);
    }

    /**
     * Старый запасной вариант:
     * /local/mvc/?route=/about
     */
    $routeFromGet = trim((string)$this->get('route', ''));

    if ($routeFromGet !== '') {
        return $this->normalizePath($routeFromGet);
    }

    // дальше оставь старый код метода
}


---

Проверка

После этого проверь:

https://bitrix24-stage.gaz.ru/local/mvc/?route=/about

Должно работать.

Потом:

https://bitrix24-stage.gaz.ru/local/mvc/about

Тоже должно работать.


---

Почему так произошло простыми словами

Сейчас браузер говорит серверу:

Дай мне /local/mvc/about

А сервер отвечает:

Такого файла нет.
Отдам это Битриксу как 404.

А нам нужно правило:

Если адрес начинается с /local/mvc/,
всегда отправляй его в /local/mvc/index.php.

И уже наш MVC внутри сам решит:

/about → HomeController::about()
/ping  → HomeController::ping()

Для Битрикса самый мягкий вариант — через /urlrewrite.php, без правки глобального конфига angie/nginx.