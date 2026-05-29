Да, ошибка простая:

Call to undefined function mvc_e()

Значит PHP дошёл до view:

/local/mvc_demo/Views/form/index.php

увидел:

mvc_e(...)

но функция mvc_e() не была подключена.

Чиним.


---

1. Создай или проверь файл /local/mvc/helpers.php

Файл должен быть именно здесь:

/srv/bx/docroot/local/mvc/helpers.php

Код полностью:

<?php

use Local\Mvc\Core\App;

if (!function_exists('mvc_route')) {
    function mvc_route(string $name, array $params = [], array $query = []): string
    {
        return App::route($name, $params, $query);
    }
}

if (!function_exists('mvc_e')) {
    function mvc_e(mixed $value): string
    {
        if (function_exists('htmlspecialcharsbx')) {
            return htmlspecialcharsbx((string)$value);
        }

        return htmlspecialchars((string)$value, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
    }
}

Важно: в этом файле не должно быть:

namespace ...

Функции должны быть глобальные.


---

2. Замени /local/mvc/bootstrap.php

Полностью замени файл:

<?php

/**
 * bootstrap.php
 *
 * Подключает Битрикс, автозагрузку MVC и helper-функции.
 */

require_once $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

spl_autoload_register(function ($class) {
    $map = [
        'Local\\Mvc\\' => $_SERVER['DOCUMENT_ROOT'] . '/local/mvc/',
    ];

    if (
        defined('LOCAL_MVC_PROJECT_NAMESPACE')
        && defined('LOCAL_MVC_PROJECT_ROOT')
    ) {
        $projectNamespace = rtrim((string)LOCAL_MVC_PROJECT_NAMESPACE, '\\') . '\\';
        $projectRoot = rtrim((string)LOCAL_MVC_PROJECT_ROOT, '/');

        $map[$projectNamespace] = $projectRoot . '/';
    }

    foreach ($map as $prefix => $baseDir) {
        if (strncmp($prefix, $class, strlen($prefix)) !== 0) {
            continue;
        }

        $relativeClass = substr($class, strlen($prefix));
        $file = rtrim($baseDir, '/') . '/' . str_replace('\\', '/', $relativeClass) . '.php';

        if (is_file($file)) {
            require_once $file;
        }

        return;
    }
});

/**
 * Helper-функции для views:
 * mvc_e()
 * mvc_route()
 */
$helpersFile = $_SERVER['DOCUMENT_ROOT'] . '/local/mvc/helpers.php';

if (is_file($helpersFile)) {
    require_once $helpersFile;
}


---

3. Сбрось OPcache

В PHP-командной строке Битрикса:

opcache_reset();
echo 'OPcache reset OK';


---

4. Проверь

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/form

Ошибка должна пропасть.


---

Если всё равно будет ошибка

Тогда временно в /local/mvc_demo/index.php после строки:

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/mvc/bootstrap.php';

добавь проверку:

if (!function_exists('mvc_e')) {
    die('mvc_e не подключился. Проверь /local/mvc/helpers.php и bootstrap.php');
}

Но скорее всего достаточно создать helpers.php, обновить bootstrap.php и сбросить OPcache.