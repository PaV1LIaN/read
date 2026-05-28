Отлично. Дальше делаем старые значения формы через Flash.

Сейчас при ошибке у нас так:

POST /form/send
  ↓
есть ошибки
  ↓
сразу показываем форму с ошибками

Работает, но правильнее будет так:

POST /form/send
  ↓
есть ошибки
  ↓
кладём ошибки и старые значения во Flash
  ↓
redirect обратно на /form
  ↓
GET /form показывает ошибки и возвращает введённые значения

Это называется нормальная схема:

POST → Redirect → GET


---

1. Заменяем /local/mvc/Core/Flash.php

Полностью замени файл:

/local/mvc/Core/Flash.php

на этот:

<?php

namespace Local\Mvc\Core;

/**
 * Flash
 *
 * Одноразовые данные в сессии.
 *
 * Используем для:
 * - сообщений
 * - старых значений формы
 */
class Flash
{
    private const MESSAGES_KEY = 'LOCAL_MVC_FLASH_MESSAGES';
    private const OLD_KEY = 'LOCAL_MVC_FLASH_OLD';

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

        if (!isset($_SESSION[self::MESSAGES_KEY])) {
            $_SESSION[self::MESSAGES_KEY] = [];
        }

        $_SESSION[self::MESSAGES_KEY][] = [
            'type' => $type,
            'message' => $message,
        ];
    }

    /**
     * Забрать сообщения и удалить их.
     */
    public static function all(): array
    {
        self::ensureSession();

        $messages = $_SESSION[self::MESSAGES_KEY] ?? [];

        unset($_SESSION[self::MESSAGES_KEY]);

        return is_array($messages) ? $messages : [];
    }

    /**
     * Сохранить старые значения формы.
     */
    public static function old(array $data): void
    {
        self::ensureSession();

        $_SESSION[self::OLD_KEY] = $data;
    }

    /**
     * Забрать старые значения формы и удалить их.
     */
    public static function getOld(): array
    {
        self::ensureSession();

        $old = $_SESSION[self::OLD_KEY] ?? [];

        unset($_SESSION[self::OLD_KEY]);

        return is_array($old) ? $old : [];
    }

    private static function ensureSession(): void
    {
        if (session_status() === PHP_SESSION_NONE) {
            session_start();
        }
    }
}


---

2. Обновляем /local/mvc/Core/Controller.php

В методе render() у нас уже есть:

$flash = Flash::all();

Сразу после этой строки добавь:

$oldFromFlash = Flash::getOld();

if (!isset($old) || !is_array($old)) {
    $old = [];
}

$old = array_replace($old, $oldFromFlash);

Должно получиться примерно так:

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

Что это значит

Если контроллер передал:

'old' => [
    'name' => '',
    'message' => '',
]

но во Flash лежит:

[
    'name' => 'Иван',
    'message' => 'Привет'
]

то во view попадёт:

$old['name'] = 'Иван';
$old['message'] = 'Привет';


---

3. Обновляем /local/mvc_demo/Controllers/FormController.php

Теперь при ошибках не будем сразу показывать форму.
Будем сохранять ошибки и старые значения во Flash, потом делать redirect.

Полностью замени файл:

/local/mvc_demo/Controllers/FormController.php

на:

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
         * Если есть ошибки:
         * 1. Сохраняем старые значения.
         * 2. Кладём ошибки во Flash.
         * 3. Возвращаем пользователя обратно на форму.
         */
        if (!empty($errors)) {
            Flash::old($data);

            foreach ($errors as $error) {
                Flash::error($error);
            }

            return $this->redirect('/local/mvc_demo/form');
        }

        /**
         * Если всё хорошо:
         * 1. Кладём сообщение об успехе.
         * 2. Возвращаем пользователя обратно на форму.
         */
        Flash::success('Форма успешно отправлена. Имя: ' . $data['name']);

        return $this->redirect('/local/mvc_demo/form');
    }
}


---

4. Обновляем /local/mvc_demo/Views/form/index.php

Теперь блок $errors можно убрать, потому что ошибки будут приходить через $flash.

Но чтобы не усложнять, можно оставить как есть. Главное, чтобы сверху был блок:

<?php if (!empty($flash)): ?>

Он уже показывает и success, и error.

Если хочешь сделать чище, удали этот старый кусок:

<?php if (!empty($errors)): ?>
    ...
<?php endif; ?>

Он теперь почти не нужен.


---

5. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/form

Проверь так:

1. В поле "Имя" введи: А
2. В поле "Сообщение" введи: 123
3. Нажми "Отправить"

Должно произойти:

1. POST ушёл на /form/send.
2. Контроллер нашёл ошибки.
3. Вернул тебя обратно на /form.
4. Показались красные ошибки.
5. В полях остались введённые значения.

Потом введи нормальные данные и отправь.

Должно быть:

1. Зелёное сообщение сверху.
2. Поля пустые.
3. При F5 сообщение исчезает.


---

Что мы сделали

Теперь у нас есть нормальная схема формы:

GET /form
  показывает форму

POST /form/send
  проверяет данные

если ошибка:
  Flash::old($data)
  Flash::error(...)
  redirect /form

если успех:
  Flash::success(...)
  redirect /form

Это очень важный кирпич для будущих проектов.

Потом в sitebuilder будет так:

Flash::success('Сайт успешно создан');
return $this->redirect('/local/sitebuilder/sites');

или при ошибке:

Flash::old($data);
Flash::error('Введите название сайта');
return $this->redirect('/local/sitebuilder/sites/create');

Дальше логичный шаг — сделать redirectBack() и backUrl, чтобы не писать руками /local/mvc_demo/form.