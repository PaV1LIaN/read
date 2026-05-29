Идём дальше. Сейчас сделаем поддержку PUT / PATCH / DELETE через _method.

Зачем это нужно:

GET     — открыть страницу
POST    — создать
PUT     — полностью обновить
PATCH   — частично обновить
DELETE  — удалить

Но обычная HTML-форма умеет только:

GET
POST

Поэтому во фреймворках часто делают так:

<form method="post">
    <input type="hidden" name="_method" value="DELETE">
</form>

Форма физически отправляет POST, но наш MVC понимает это как DELETE.


---

1. Обнови /local/mvc/Core/Request.php

Найди метод:

public function method(): string

и замени его на:

public function method(): string
{
    $method = strtoupper((string)$this->server('REQUEST_METHOD', 'GET'));

    /**
     * HTML-форма не умеет DELETE/PUT/PATCH.
     * Поэтому разрешаем подмену метода через скрытое поле _method.
     *
     * Пример:
     * <input type="hidden" name="_method" value="DELETE">
     */
    if ($method === 'POST') {
        $override = strtoupper(trim((string)$this->post('_method', '')));

        if ($override === '') {
            $override = strtoupper(trim((string)$this->header('X-HTTP-Method-Override', '')));
        }

        if (in_array($override, ['PUT', 'PATCH', 'DELETE'], true)) {
            return $override;
        }
    }

    return $method;
}

Если метода header() ещё нет, добавь в Request.php:

public function header(string $name, mixed $default = null): mixed
{
    $key = 'HTTP_' . strtoupper(str_replace('-', '_', $name));

    return $this->server[$key] ?? $default;
}


---

2. Обнови /local/mvc/Core/Router.php

Внутри класса Router после методов get() и post() добавь:

public function put(string $path, array $handler, array $middleware = [], ?string $name = null): void
{
    $this->add('PUT', $path, $handler, $middleware, $name);
}

public function patch(string $path, array $handler, array $middleware = [], ?string $name = null): void
{
    $this->add('PATCH', $path, $handler, $middleware, $name);
}

public function delete(string $path, array $handler, array $middleware = [], ?string $name = null): void
{
    $this->add('DELETE', $path, $handler, $middleware, $name);
}

Теперь можно будет писать:

$router->delete('/something/delete', [Controller::class, 'delete'], ['csrf']);


---

3. Обнови CSRF в /local/mvc/Core/Middleware.php

Найди блок:

if ($name === 'csrf') {

и замени его полностью на:

if ($name === 'csrf') {
    /**
     * Безопасные методы не проверяем.
     */
    if (in_array($request->method(), ['GET', 'HEAD', 'OPTIONS'], true)) {
        return null;
    }

    /**
     * Обычная форма Битрикса через bitrix_sessid_post().
     */
    if (function_exists('check_bitrix_sessid') && check_bitrix_sessid()) {
        return null;
    }

    /**
     * AJAX/JSON-запрос через заголовок.
     */
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

Важно: раньше CSRF проверялся только для POST. Теперь он проверяется для всех опасных методов:

POST
PUT
PATCH
DELETE


---

4. Создай контроллер /local/mvc_demo/Controllers/MethodDemoController.php

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Flash;
use Local\Mvc\Core\Response;

class MethodDemoController extends Controller
{
    public function index(): Response
    {
        return $this->render('method/index', [
            'title' => 'Method Demo',
        ]);
    }

    public function delete(): Response
    {
        Flash::success('DELETE-запрос успешно обработан через _method.');

        return $this->redirectRoute('method.index');
    }
}


---

5. Создай view /local/mvc_demo/Views/method/index.php

Сначала создай папку:

/local/mvc_demo/Views/method/

Файл:

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

?>

<div class="mvc-card">
    <h1 class="mvc-page-title">
        <?= htmlspecialcharsbx($title ?? 'Method Demo') ?>
    </h1>

    <p class="mvc-page-text">
        Эта страница проверяет DELETE-запрос через скрытое поле
        <span class="mvc-code">_method</span>.
    </p>

    <?php if (!empty($flash)): ?>
        <?php foreach ($flash as $item): ?>
            <div class="mvc-info" style="border-color: #bbf7d0; background: #f0fdf4; color: #166534;">
                <?= htmlspecialcharsbx($item['message'] ?? '') ?>
            </div>
        <?php endforeach; ?>
    <?php endif; ?>

    <div class="mvc-info">
        <b>Как это работает:</b>

        <ol>
            <li>Форма отправляется обычным методом <span class="mvc-code">POST</span>.</li>
            <li>Внутри формы есть поле <span class="mvc-code">_method = DELETE</span>.</li>
            <li>Request превращает POST в DELETE.</li>
            <li>Router находит DELETE-маршрут.</li>
            <li>Controller обрабатывает удаление.</li>
        </ol>
    </div>

    <form method="post" action="<?= mvc_e(mvc_route('method.delete')) ?>" style="margin-top: 24px;">
        <?php if (function_exists('bitrix_sessid_post')): ?>
            <?= bitrix_sessid_post() ?>
        <?php endif; ?>

        <input type="hidden" name="_method" value="DELETE">

        <button
            type="submit"
            style="min-height: 42px; padding: 0 18px; border: 0; border-radius: 10px; background: #dc2626; color: #fff; font-weight: 600; cursor: pointer;"
        >
            Проверить DELETE
        </button>
    </form>
</div>


---

6. Обнови /local/mvc_demo/routes.php

Добавь use:

use Local\MvcDemo\Controllers\MethodDemoController;

В публичные маршруты добавь:

$router->get('/method-demo', [MethodDemoController::class, 'index'], [], 'method.index');

$router->delete('/method-demo/delete', [MethodDemoController::class, 'delete'], ['csrf'], 'method.delete');


---

7. Добавь ссылку в меню

Файл:

/local/mvc_demo/Views/layouts/app.php

В меню добавь:

<a href="/local/mvc_demo/method-demo">Methods</a>


---

8. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/method-demo

Нажми:

Проверить DELETE

Должно вернуть обратно на страницу и показать зелёное сообщение:

DELETE-запрос успешно обработан через _method.


---

Что мы добавили:

$router->put(...)
$router->patch(...)
$router->delete(...)

Теперь для будущих проектов можно писать красиво:

$router->post('/sites', [SiteController::class, 'store'], ['csrf'], 'sites.store');

$router->put('/sites/{siteId:\d+}', [SiteController::class, 'update'], ['csrf'], 'sites.update');

$router->delete('/sites/{siteId:\d+}', [SiteController::class, 'delete'], ['csrf'], 'sites.delete');

Это уже ближе к нормальному CRUD-фреймворку.