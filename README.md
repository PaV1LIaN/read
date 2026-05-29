Да, идём дальше в сторону Laravel.

Сейчас сделаем Laravel-like helper-функции для шаблонов:

route('form.send')
old('name')
csrf_field()
method_field('DELETE')
e($value)

То есть во view будет почти как в Laravel.


---

1. Создай /local/mvc/Core/ViewData.php

Это маленькое хранилище данных текущего view.

<?php

namespace Local\Mvc\Core;

/**
 * ViewData
 *
 * Хранилище данных текущего шаблона.
 *
 * Нужно, чтобы helper old('name') мог достать старое значение формы.
 */
class ViewData
{
    private static array $data = [];

    public static function set(array $data): void
    {
        self::$data = $data;
    }

    public static function get(string $key, mixed $default = null): mixed
    {
        return self::$data[$key] ?? $default;
    }

    public static function old(string $key, mixed $default = null): mixed
    {
        $old = self::get('old', []);

        if (!is_array($old)) {
            return $default;
        }

        return $old[$key] ?? $default;
    }
}


---

2. Обнови /local/mvc/Core/Controller.php

В методе render() найди место, где у тебя уже есть:

extract($params);

$flash = Flash::all();

$oldFromFlash = Flash::getOld();

if (!isset($old) || !is_array($old)) {
    $old = [];
}

$old = array_replace($old, $oldFromFlash);

Сразу после этого добавь:

ViewData::set(array_merge($params, [
    'flash' => $flash,
    'old' => $old,
]));

Должно получиться так:

extract($params);

/**
 * Flash-сообщения.
 */
$flash = Flash::all();

/**
 * Старые значения формы.
 */
$oldFromFlash = Flash::getOld();

if (!isset($old) || !is_array($old)) {
    $old = [];
}

$old = array_replace($old, $oldFromFlash);

/**
 * Данные для Laravel-like helper-функций:
 * old('name')
 */
ViewData::set(array_merge($params, [
    'flash' => $flash,
    'old' => $old,
]));


---

3. Замени /local/mvc/helpers.php

Полностью замени файл:

<?php

use Local\Mvc\Core\App;
use Local\Mvc\Core\ViewData;

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

/**
 * Laravel-like route()
 *
 * Пример:
 * route('admin.users.show', ['id' => 5])
 */
if (!function_exists('route')) {
    function route(string $name, array $params = [], array $query = []): string
    {
        return mvc_route($name, $params, $query);
    }
}

/**
 * Laravel-like e()
 *
 * Пример:
 * e($title)
 */
if (!function_exists('e')) {
    function e(mixed $value): string
    {
        return mvc_e($value);
    }
}

/**
 * Laravel-like old()
 *
 * Пример:
 * old('name')
 */
if (!function_exists('old')) {
    function old(string $key, mixed $default = ''): mixed
    {
        return ViewData::old($key, $default);
    }
}

/**
 * Laravel-like csrf_field()
 *
 * Пример:
 * <?= csrf_field() ?>
 */
if (!function_exists('csrf_field')) {
    function csrf_field(): string
    {
        if (function_exists('bitrix_sessid_post')) {
            return bitrix_sessid_post();
        }

        return '';
    }
}

/**
 * Laravel-like method_field()
 *
 * Пример:
 * <?= method_field('DELETE') ?>
 */
if (!function_exists('method_field')) {
    function method_field(string $method): string
    {
        return '<input type="hidden" name="_method" value="' . e(strtoupper($method)) . '">';
    }
}


---

4. Обнови /local/mvc_demo/Views/form/index.php

Найди форму.

Было примерно так:

<form method="post" action="<?= mvc_e(mvc_route('form.send')) ?>" style="margin-top: 24px;">
    <?php if (function_exists('bitrix_sessid_post')): ?>
        <?= bitrix_sessid_post() ?>
    <?php endif; ?>

Замени на Laravel-like вариант:

<form method="post" action="<?= e(route('form.send')) ?>" style="margin-top: 24px;">
    <?= csrf_field() ?>

Теперь найди поле name.

Было:

value="<?= htmlspecialcharsbx($formName) ?>"

Замени на:

value="<?= e(old('name')) ?>"

Найди textarea.

Было:

><?= htmlspecialcharsbx($formMessage) ?></textarea>

Замени на:

><?= e(old('message')) ?></textarea>


---

5. Обнови /local/mvc_demo/Views/method/index.php

Найди форму DELETE.

Было:

<form method="post" action="<?= mvc_e(mvc_route('method.delete')) ?>" style="margin-top: 24px;">
    <?php if (function_exists('bitrix_sessid_post')): ?>
        <?= bitrix_sessid_post() ?>
    <?php endif; ?>

    <input type="hidden" name="_method" value="DELETE">

Замени на:

<form method="post" action="<?= e(route('method.delete')) ?>" style="margin-top: 24px;">
    <?= csrf_field() ?>
    <?= method_field('DELETE') ?>


---

6. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Проверь форму:

/local/mvc_demo/form

Проверь:

1. Отправь пустую форму.
2. Ошибки должны появиться.
3. Введённые значения должны сохраняться.
4. Отправь нормальные данные.

Проверь method demo:

/local/mvc_demo/method-demo

Нажми:

Проверить DELETE


---

Что мы сделали

Теперь шаблоны можно писать ближе к Laravel:

<form method="post" action="<?= e(route('form.send')) ?>">
    <?= csrf_field() ?>

    <input name="name" value="<?= e(old('name')) ?>">
</form>

И DELETE-форма:

<form method="post" action="<?= e(route('method.delete')) ?>">
    <?= csrf_field() ?>
    <?= method_field('DELETE') ?>

    <button>Удалить</button>
</form>

Главная мысль:

route()        — собрать URL по имени маршрута
old()          — вернуть старое значение поля после ошибки
csrf_field()   — вставить sessid Битрикса
method_field() — подменить POST на DELETE / PUT / PATCH
e()            — безопасно вывести текст

Это уже очень похоже на Laravel, но адаптировано под Битрикс24.