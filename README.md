Идём дальше. Сделаем простые helper-функции для views, но без большого View-класса, чтобы не путаться.

Зачем:

Сейчас во view приходится писать длинно:

\Local\Mvc\Core\App::route('admin.users.show', ['id' => 5])

А хотим коротко:

mvc_route('admin.users.show', ['id' => 5])


---

1. Создай файл /local/mvc/helpers.php

<?php

use Local\Mvc\Core\App;

if (!function_exists('mvc_route')) {
    /**
     * Собрать URL по имени маршрута.
     *
     * Пример:
     * mvc_route('admin.users.show', ['id' => 5])
     */
    function mvc_route(string $name, array $params = [], array $query = []): string
    {
        return App::route($name, $params, $query);
    }
}

if (!function_exists('mvc_e')) {
    /**
     * Безопасный вывод текста.
     *
     * Это короткая замена htmlspecialcharsbx().
     */
    function mvc_e(mixed $value): string
    {
        if (function_exists('htmlspecialcharsbx')) {
            return htmlspecialcharsbx((string)$value);
        }

        return htmlspecialchars((string)$value, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
    }
}


---

2. Подключи helper в /local/mvc/bootstrap.php

Открой:

/local/mvc/bootstrap.php

После автозагрузчика добавь:

$helpersFile = $_SERVER['DOCUMENT_ROOT'] . '/local/mvc/helpers.php';

if (is_file($helpersFile)) {
    require_once $helpersFile;
}

Полностью конец файла должен выглядеть примерно так:

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

$helpersFile = $_SERVER['DOCUMENT_ROOT'] . '/local/mvc/helpers.php';

if (is_file($helpersFile)) {
    require_once $helpersFile;
}


---

3. Обнови ссылки в /local/mvc_demo/Views/admin/users.php

Найди ссылку:

<a href="<?= htmlspecialcharsbx(\Local\Mvc\Core\App::route('admin.users.show', [
    'id' => (int)($user['id'] ?? 0),
])) ?>">
    Открыть
</a>

Замени на короткую:

<a href="<?= mvc_e(mvc_route('admin.users.show', [
    'id' => (int)($user['id'] ?? 0),
])) ?>">
    Открыть
</a>


---

4. Обнови ссылку назад в /local/mvc_demo/Views/admin/user_detail.php

Найди:

<a href="<?= htmlspecialcharsbx(\Local\Mvc\Core\App::route('admin.users.index')) ?>">
    ← Назад к списку
</a>

Замени на:

<a href="<?= mvc_e(mvc_route('admin.users.index')) ?>">
    ← Назад к списку
</a>


---

5. Обнови форму /local/mvc_demo/Views/form/index.php

Найди:

<form method="post" action="/local/mvc_demo/form/send" style="margin-top: 24px;">

Замени на:

<form method="post" action="<?= mvc_e(mvc_route('form.send')) ?>" style="margin-top: 24px;">

Теперь форма тоже не знает точный URL.


---

6. Проверь, что маршруты названы

В /local/mvc_demo/routes.php должно быть:

$router->get('/form', [FormController::class, 'index'], [], 'form.index');

$router->post('/form/send', [FormController::class, 'send'], ['csrf'], 'form.send');

И в админской группе:

$router->get('/users', [AdminController::class, 'users'], [], 'admin.users.index');

$router->get('/users/{id:\d+}', [AdminController::class, 'userDetail'], [], 'admin.users.show');


---

7. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Проверь:

/local/mvc_demo/form
/local/mvc_demo/admin/users
/local/mvc_demo/admin/users/1
/local/mvc_demo/debug/routes


---

Что мы сделали

Теперь во view можно писать коротко:

mvc_route('admin.users.show', ['id' => 5])

вместо длинного:

\Local\Mvc\Core\App::route('admin.users.show', ['id' => 5])

И можно безопасно выводить текст так:

mvc_e($value)

вместо:

htmlspecialcharsbx($value)

Главная мысль:

App::route() — ядро фреймворка.
mvc_route() — удобная короткая функция для views.

Дальше логично сделать named route group prefix, чтобы маршруты внутри /admin автоматически получали имена типа admin.users.index, а не писать полное имя руками каждый раз.