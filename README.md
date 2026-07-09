Идём дальше. Сейчас сделаем Laravel-like view() helper для partials.

Сейчас у нас в index.php и trash.php некрасиво:

require $_SERVER['DOCUMENT_ROOT'] . '/local/mvc_demo/Views/partials/pagination.php';

Сделаем красиво, как в Laravel:

<?= view('partials.pagination', [
    'pagination' => $pagination,
    'routeName' => 'notes.index',
]) ?>


---

1. Создай файл /local/mvc/Core/View.php

<?php

namespace Local\Mvc\Core;

use RuntimeException;

class View
{
    /**
     * Laravel-like render view.
     *
     * Пример:
     * View::render('partials.pagination', ['pagination' => $pagination])
     */
    public static function render(string $view, array $data = []): string
    {
        $path = self::path($view);

        if (!is_file($path)) {
            throw new RuntimeException('VIEW_NOT_FOUND: ' . $path);
        }

        ob_start();

        extract($data, EXTR_SKIP);

        require $path;

        return (string)ob_get_clean();
    }

    public static function exists(string $view): bool
    {
        return is_file(self::path($view));
    }

    public static function path(string $view): string
    {
        $view = trim($view);

        if ($view === '') {
            throw new RuntimeException('VIEW_NAME_IS_EMPTY');
        }

        /**
         * Поддерживаем два варианта:
         *
         * partials.pagination
         * partials/pagination
         */
        $view = str_replace('.', '/', $view);
        $view = trim($view, '/');

        if (!defined('LOCAL_MVC_PROJECT_ROOT')) {
            throw new RuntimeException('LOCAL_MVC_PROJECT_ROOT_NOT_DEFINED');
        }

        return rtrim(LOCAL_MVC_PROJECT_ROOT, '/') . '/Views/' . $view . '.php';
    }
}


---

2. Обнови /local/mvc/helpers.php

В конец файла добавь:

if (!function_exists('view')) {
    /**
     * Laravel-like view() helper.
     *
     * Пример:
     * echo view('partials.pagination', [
     *     'pagination' => $pagination,
     * ]);
     */
    function view(string $view, array $data = []): string
    {
        return \Local\Mvc\Core\View::render($view, $data);
    }
}

if (!function_exists('view_exists')) {
    function view_exists(string $view): bool
    {
        return \Local\Mvc\Core\View::exists($view);
    }
}

Теперь можно писать:

<?= view('partials.pagination', [...]) ?>


---

3. Обнови подключение пагинации в notes/index.php

Внизу файла найди старый блок:

<?php
$routeName = 'notes.index';
$routeParams = [];
$query = [];

if (!empty($search)) {
    $query['q'] = $search;
}

require $_SERVER['DOCUMENT_ROOT'] . '/local/mvc_demo/Views/partials/pagination.php';
?>

Замени на:

<?php
$query = [];

if (!empty($search)) {
    $query['q'] = $search;
}
?>

<?= view('partials.pagination', [
    'pagination' => $pagination ?? [],
    'routeName' => 'notes.index',
    'routeParams' => [],
    'query' => $query,
]) ?>


---

4. Обнови подключение пагинации в notes/trash.php

Внизу файла найди старый блок:

<?php
$routeName = 'notes.trash';
$routeParams = [];
$query = [];

if (!empty($search)) {
    $query['q'] = $search;
}

require $_SERVER['DOCUMENT_ROOT'] . '/local/mvc_demo/Views/partials/pagination.php';
?>

Замени на:

<?php
$query = [];

if (!empty($search)) {
    $query['q'] = $search;
}
?>

<?= view('partials.pagination', [
    'pagination' => $pagination ?? [],
    'routeName' => 'notes.trash',
    'routeParams' => [],
    'query' => $query,
]) ?>


---

5. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/notes

Потом:

/local/mvc_demo/notes/trash

Пагинация должна отображаться как раньше.


---

Что мы сделали

Было так:

require $_SERVER['DOCUMENT_ROOT'] . '/local/mvc_demo/Views/partials/pagination.php';

Стало так:

<?= view('partials.pagination', [
    'pagination' => $pagination,
]) ?>

Главная мысль:

view() — это Laravel-like способ подключать маленькие части интерфейса.

Теперь мы можем делать переиспользуемые partials:

partials.pagination
partials.alert
partials.errors
partials.form-input

И не писать каждый раз длинные пути через $_SERVER['DOCUMENT_ROOT'].