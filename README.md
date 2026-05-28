Давай дальше. Сейчас делаем Flash-сообщения.

Зачем это нужно

Сейчас форма работает так:

GET  /form       — показать форму
POST /form/send  — обработать форму и показать результат

Но это не идеально.

Почему?

Если пользователь после успешной отправки нажмёт F5, браузер может спросить:

Повторно отправить форму?

И POST-запрос может выполниться ещё раз.

Правильная схема такая:

GET  /form
  ↓
показали форму

POST /form/send
  ↓
проверили данные
  ↓
записали flash-сообщение
  ↓
redirect обратно на /form

GET /form
  ↓
показали сообщение "Форма успешно отправлена"

То есть после успешного POST мы делаем редирект.


---

1. Создай /local/mvc/Core/Flash.php

<?php

namespace Local\Mvc\Core;

/**
 * Flash
 *
 * Одноразовые сообщения.
 *
 * Простыми словами:
 * мы кладём сообщение в сессию,
 * показываем его на следующей странице,
 * и сразу удаляем.
 */
class Flash
{
    private const SESSION_KEY = 'LOCAL_MVC_FLASH';

    public static function success(string $message): void
    {
        self::add('success', $message);
    }

    public static function error(string $message): void
    {
        self::add('error', $message);
    }

    public static function warning(string $message): void
    {
        self::add('warning', $message);
    }

    public static function info(string $message): void
    {
        self::add('info', $message);
    }

    public static function add(string $type, string $message): void
    {
        self::ensureSession();

        if (!isset($_SESSION[self::SESSION_KEY])) {
            $_SESSION[self::SESSION_KEY] = [];
        }

        $_SESSION[self::SESSION_KEY][] = [
            'type' => $type,
            'message' => $message,
        ];
    }

    /**
     * Забрать все сообщения и удалить их.
     */
    public static function all(): array
    {
        self::ensureSession();

        $messages = $_SESSION[self::SESSION_KEY] ?? [];

        unset($_SESSION[self::SESSION_KEY]);

        return is_array($messages) ? $messages : [];
    }

    private static function ensureSession(): void
    {
        if (session_status() === PHP_SESSION_NONE) {
            session_start();
        }
    }
}


---

2. Обнови /local/mvc/Core/Controller.php

Нужно, чтобы каждый View автоматически получал переменную:

$flash

Открой файл:

/local/mvc/Core/Controller.php

Найди в методе render() строку:

extract($params);

Сразу после неё добавь:

$flash = Flash::all();

Должно быть так:

extract($params);

/**
 * Flash-сообщения.
 *
 * Они доступны в любом View через переменную $flash.
 */
$flash = Flash::all();

Теперь любой шаблон сможет показать сообщения.


---

3. Обнови /local/mvc_demo/Views/form/index.php

В этом файле после описания формы добавим вывод Flash-сообщений.

Найди место после текста:

<p class="mvc-page-text">
    Это простая форма, чтобы проверить POST-запросы в нашем MVC.
</p>

Сразу после него вставь:

<?php if (!empty($flash)): ?>
    <?php foreach ($flash as $item): ?>
        <?php
        $type = $item['type'] ?? 'info';
        $message = $item['message'] ?? '';

        $style = 'border-color: #bfdbfe; background: #eff6ff; color: #1d4ed8;';

        if ($type === 'success') {
            $style = 'border-color: #bbf7d0; background: #f0fdf4; color: #166534;';
        } elseif ($type === 'error') {
            $style = 'border-color: #fecaca; background: #fef2f2; color: #991b1b;';
        } elseif ($type === 'warning') {
            $style = 'border-color: #fde68a; background: #fffbeb; color: #92400e;';
        }
        ?>

        <div class="mvc-info" style="<?= htmlspecialcharsbx($style) ?>">
            <?= htmlspecialcharsbx($message) ?>
        </div>
    <?php endforeach; ?>
<?php endif; ?>


---

4. Обнови /local/mvc_demo/Controllers/FormController.php

Теперь при успешной отправке будем не рендерить страницу сразу, а делать flash + redirect.

Полностью замени FormController.php:

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Flash;
use Local\Mvc\Core\Response;
use Local\Mvc\Core\Validator;

class FormController extends Controller
{
    /**
     * Показать форму.
     */
    public function index(): Response
    {
        return $this->render('form/index', [
            'title' => 'Тестовая форма',
            'errors' => [],
            'success' => '',
            'old' => [
                'name' => '',
                'message' => '',
            ],
        ]);
    }

    /**
     * Обработать отправку формы.
     */
    public function send(): Response
    {
        $data = [
            'name' => trim((string)$this->request->post('name', '')),
            'message' => trim((string)$this->request->post('message', '')),
        ];

        $errors = [];

        if (function_exists('check_bitrix_sessid') && !check_bitrix_sessid()) {
            $errors[] = 'Ошибка безопасности: неверный sessid.';
        }

        $validator = Validator::make($data)
            ->required('name', 'Введите имя.')
            ->min('name', 2, 'Имя должно быть не короче 2 символов.')
            ->max('name', 100, 'Имя должно быть не длиннее 100 символов.')
            ->required('message', 'Введите сообщение.')
            ->min('message', 5, 'Сообщение должно быть не короче 5 символов.')
            ->max('message', 1000, 'Сообщение должно быть не длиннее 1000 символов.');

        if ($validator->fails()) {
            $errors = array_merge($errors, $validator->errorList());
        }

        /**
         * Если ошибки есть — остаёмся на этой же странице
         * и показываем ошибки.
         */
        if (!empty($errors)) {
            return $this->render('form/index', [
                'title' => 'Тестовая форма',
                'errors' => $errors,
                'success' => '',
                'old' => $data,
            ]);
        }

        /**
         * Если всё хорошо:
         * 1. Кладём одноразовое сообщение.
         * 2. Делаем redirect обратно на форму.
         */
        Flash::success('Форма успешно отправлена. Имя: ' . $data['name']);

        return $this->redirect('/local/mvc_demo/form');
    }
}


---

5. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/form

Заполни форму нормально и отправь.

Должно произойти так:

1. POST уйдёт на /form/send.
2. Контроллер проверит данные.
3. Положит сообщение во Flash.
4. Перекинет обратно на /form.
5. На /form появится зелёное сообщение.

После этого обнови страницу F5.

Сообщение должно исчезнуть.

Это и есть смысл Flash:

показать один раз и забыть


---

Что мы сделали

Добавили новый общий инструмент:

Flash::success('Готово');
Flash::error('Ошибка');
Flash::warning('Внимание');
Flash::info('Информация');

Теперь его можно будет использовать в любых проектах:

sitebuilder
glab
qr_opros
admin dashboard

Например:

Flash::success('Сайт успешно создан');
return $this->redirect('/local/sitebuilder/sites');

Или:

Flash::error('Недостаточно прав');
return $this->redirect('/local/sitebuilder/');

Главная мысль:

POST делает действие.
Flash запоминает сообщение.
Redirect возвращает пользователя на нормальную страницу.
GET показывает результат.

Следующий шаг — сделать Request old input, чтобы при ошибках после redirect можно было вернуть старые введённые значения.