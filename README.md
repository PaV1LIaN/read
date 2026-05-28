Идём дальше. Сейчас сделаем важную вещь для будущих проектов: AJAX-запросы к API.

Простыми словами:

HTML-страница показывает кнопку.
JS нажимает кнопку.
fetch отправляет POST-запрос в API.
API возвращает JSON.
Страница показывает результат без перезагрузки.

Это потом пригодится для:

sitebuilder — сохранить блок без перезагрузки
диск — создать папку через AJAX
glab — сменить статус заявки
qr — фильтры и графики


---

1. Добавим работу с headers в Request

Открой файл:

/local/mvc/Core/Request.php

Внутрь класса добавь метод:

/**
 * Получить HTTP-заголовок.
 *
 * Например:
 * $request->header('X-Bitrix-Sessid')
 */
public function header(string $name, mixed $default = null): mixed
{
    $key = 'HTTP_' . strtoupper(str_replace('-', '_', $name));

    return $this->server[$key] ?? $default;
}

Добавь его рядом с методом:

public function server(string $key, mixed $default = null): mixed


---

2. Обновим csrf middleware

Сейчас csrf хорошо работает для обычной формы, но для JSON/AJAX удобнее передавать sessid в заголовке:

X-Bitrix-Sessid: ...

Открой:

/local/mvc/Core/Middleware.php

Найди блок:

if ($middleware === 'csrf') {

и замени весь блок на этот:

if ($middleware === 'csrf') {
    if ($request->method() !== 'POST') {
        return null;
    }

    /**
     * Вариант 1:
     * обычная форма Битрикса через bitrix_sessid_post().
     */
    if (function_exists('check_bitrix_sessid') && check_bitrix_sessid()) {
        return null;
    }

    /**
     * Вариант 2:
     * AJAX/JSON-запрос.
     *
     * JS может отправить:
     * X-Bitrix-Sessid: текущий_sessid
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

Теперь csrf будет работать и для обычных форм, и для AJAX.


---

3. Создаём AjaxDemoController

Создай файл:

/local/mvc_demo/Controllers/AjaxDemoController.php

Код:

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\Auth;
use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Response;

class AjaxDemoController extends Controller
{
    /**
     * Страница с AJAX-примером.
     */
    public function index(): Response
    {
        return $this->render('ajax/index', [
            'title' => 'AJAX Demo',
            'sessid' => function_exists('bitrix_sessid') ? bitrix_sessid() : '',
            'userId' => Auth::id(),
        ]);
    }

    /**
     * POST /api/ajax-demo/echo
     *
     * API принимает JSON и возвращает JSON.
     */
    public function echo(): Response
    {
        $data = $this->request->jsonAll();

        $text = trim((string)($data['text'] ?? ''));

        if ($text === '') {
            return $this->error('VALIDATION_ERROR', [
                'message' => 'Введите текст.',
            ], 422);
        }

        return $this->success([
            'received_text' => $text,
            'length' => mb_strlen($text),
            'user_id' => Auth::id(),
            'time' => date('Y-m-d H:i:s'),
        ]);
    }
}


---

4. Создаём view страницы AJAX

Создай папку:

/local/mvc_demo/Views/ajax/

Создай файл:

/local/mvc_demo/Views/ajax/index.php

Код:

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

?>

<div class="mvc-card">
    <h1 class="mvc-page-title">
        <?= htmlspecialcharsbx($title ?? 'AJAX Demo') ?>
    </h1>

    <p class="mvc-page-text">
        Эта страница отправляет JSON-запрос в API без перезагрузки страницы.
    </p>

    <div class="mvc-info">
        <b>Как это работает:</b>

        <ol>
            <li>Ты вводишь текст.</li>
            <li>JavaScript отправляет POST-запрос на <span class="mvc-code">/api/ajax-demo/echo</span>.</li>
            <li>Middleware <span class="mvc-code">csrf</span> проверяет sessid.</li>
            <li>Контроллер возвращает JSON.</li>
            <li>JS показывает результат на странице.</li>
        </ol>
    </div>

    <div style="margin-top: 24px;">
        <label style="display:block; margin-bottom: 6px; font-weight: 600;">
            Текст для отправки
        </label>

        <input
            id="ajaxText"
            type="text"
            value="Привет из AJAX"
            style="width: 100%; min-height: 42px; padding: 8px 12px; border: 1px solid #d1d5db; border-radius: 10px;"
        >
    </div>

    <div style="margin-top: 16px;">
        <button
            id="ajaxSendBtn"
            type="button"
            style="min-height: 42px; padding: 0 18px; border: 0; border-radius: 10px; background: #2563eb; color: #fff; font-weight: 600; cursor: pointer;"
        >
            Отправить AJAX
        </button>
    </div>

    <div class="mvc-info" style="margin-top: 24px;">
        <b>Ответ API:</b>

        <pre id="ajaxResult" style="white-space: pre-wrap; margin-bottom: 0;">Пока запроса не было.</pre>
    </div>
</div>

<script>
(function () {
    const button = document.getElementById('ajaxSendBtn');
    const input = document.getElementById('ajaxText');
    const result = document.getElementById('ajaxResult');

    const sessid = <?= json_encode((string)($sessid ?? ''), JSON_UNESCAPED_UNICODE) ?>;

    button.addEventListener('click', async function () {
        result.textContent = 'Отправляем запрос...';

        try {
            const response = await fetch('/local/mvc_demo/api/ajax-demo/echo', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                    'Accept': 'application/json',
                    'X-Requested-With': 'XMLHttpRequest',
                    'X-Bitrix-Sessid': sessid
                },
                body: JSON.stringify({
                    text: input.value
                })
            });

            const json = await response.json();

            result.textContent = JSON.stringify(json, null, 2);
        } catch (error) {
            result.textContent = 'AJAX ERROR: ' + error.message;
        }
    });
})();
</script>


---

5. Обновляем routes.php

Открой:

/local/mvc_demo/routes.php

Добавь use:

use Local\MvcDemo\Controllers\AjaxDemoController;

В публичные маршруты добавь страницу:

$router->get('/ajax-demo', [AjaxDemoController::class, 'index']);

В API-группу добавь POST-маршрут:

$router->post('/ajax-demo/echo', [AjaxDemoController::class, 'echo'], ['csrf']);

Полный важный кусок должен выглядеть примерно так:

use Local\MvcDemo\Controllers\AjaxDemoController;

$router->get('/ajax-demo', [AjaxDemoController::class, 'index']);

$router->group([
    'prefix' => '/api',
    'middleware' => ['auth', 'admin'],
], function (Router $router) {
    $router->get('/users', [UserApiController::class, 'index']);
    $router->get('/users/{id:\d+}', [UserApiController::class, 'show']);

    $router->post('/ajax-demo/echo', [AjaxDemoController::class, 'echo'], ['csrf']);
});

Так как API-группа защищена auth + admin, AJAX-демо будет работать только у администратора.


---

6. Добавь ссылку в меню

Файл:

/local/mvc_demo/Views/layouts/app.php

В меню добавь:

<a href="/local/mvc_demo/ajax-demo">AJAX</a>

Например:

<nav class="mvc-nav">
    <a href="/local/mvc_demo/">Главная</a>
    <a href="/local/mvc_demo/about">О проекте</a>
    <a href="/local/mvc_demo/form">Форма</a>
    <a href="/local/mvc_demo/ajax-demo">AJAX</a>
    <a href="/local/mvc_demo/me">Я</a>
    <a href="/local/mvc_demo/admin/dashboard">Админка</a>
    <a href="/local/mvc_demo/admin/users">Пользователи</a>
    <a href="/local/mvc_demo/ping" target="_blank">Ping JSON</a>
</nav>


---

7. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/ajax-demo

Нажми кнопку:

Отправить AJAX

Должен появиться JSON примерно такой:

{
  "ok": true,
  "data": {
    "received_text": "Привет из AJAX",
    "length": 15,
    "user_id": 1,
    "time": "2026-05-28 12:00:00"
  }
}


---

Что мы сделали

Теперь наш MVC умеет не только формы, но и AJAX:

Страница
  ↓
JavaScript fetch()
  ↓
POST JSON
  ↓
Router
  ↓
Middleware auth/admin/csrf
  ↓
Controller
  ↓
JSON Response
  ↓
JavaScript показывает результат

Это очень важный шаг. Именно так будут работать многие действия в реальных проектах:

сохранить блок
создать страницу
удалить файл
обновить настройки
загрузить данные графика

Дальше можно сделать отдельный ApiController, чтобы API-контроллеры имели дополнительные удобные методы для JSON-ответов.