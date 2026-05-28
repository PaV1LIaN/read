Идём дальше. Сейчас вынесем проверки формы в отдельный класс:

/local/mvc/Core/Validator.php

Простыми словами:

Controller — не должен сам проверять каждое поле вручную.
Validator — отдельный помощник, который проверяет данные.


---

1. Создай /local/mvc/Core/Validator.php

<?php

namespace Local\Mvc\Core;

/**
 * Validator
 *
 * Помощник для проверки данных.
 *
 * Например:
 * - поле обязательно
 * - минимум символов
 * - максимум символов
 */
class Validator
{
    private array $data;
    private array $errors = [];

    public function __construct(array $data)
    {
        $this->data = $data;
    }

    /**
     * Быстро создать validator.
     */
    public static function make(array $data): self
    {
        return new self($data);
    }

    /**
     * Поле обязательно.
     */
    public function required(string $field, string $message): self
    {
        $value = $this->data[$field] ?? null;

        if ($value === null || trim((string)$value) === '') {
            $this->errors[$field][] = $message;
        }

        return $this;
    }

    /**
     * Минимальная длина строки.
     */
    public function min(string $field, int $length, string $message): self
    {
        $value = trim((string)($this->data[$field] ?? ''));

        if ($value !== '' && mb_strlen($value) < $length) {
            $this->errors[$field][] = $message;
        }

        return $this;
    }

    /**
     * Максимальная длина строки.
     */
    public function max(string $field, int $length, string $message): self
    {
        $value = trim((string)($this->data[$field] ?? ''));

        if ($value !== '' && mb_strlen($value) > $length) {
            $this->errors[$field][] = $message;
        }

        return $this;
    }

    /**
     * Есть ли ошибки?
     */
    public function fails(): bool
    {
        return !empty($this->errors);
    }

    /**
     * Ошибки по полям.
     */
    public function errors(): array
    {
        return $this->errors;
    }

    /**
     * Все ошибки одним списком.
     *
     * Удобно для простого вывода во view.
     */
    public function errorList(): array
    {
        $list = [];

        foreach ($this->errors as $fieldErrors) {
            foreach ($fieldErrors as $error) {
                $list[] = $error;
            }
        }

        return $list;
    }
}


---

2. Обнови /local/mvc_demo/Controllers/FormController.php

Теперь контроллер станет чище.

Полностью замени файл:

/local/mvc_demo/Controllers/FormController.php

на:

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\Controller;
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

        /**
         * Проверяем sessid Битрикса.
         */
        if (function_exists('check_bitrix_sessid') && !check_bitrix_sessid()) {
            $errors[] = 'Ошибка безопасности: неверный sessid.';
        }

        /**
         * Проверяем поля через Validator.
         */
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

        if (!empty($errors)) {
            return $this->render('form/index', [
                'title' => 'Тестовая форма',
                'errors' => $errors,
                'success' => '',
                'old' => $data,
            ]);
        }

        return $this->render('form/index', [
            'title' => 'Тестовая форма',
            'errors' => [],
            'success' => 'Форма успешно отправлена. Имя: ' . $data['name'] . ', сообщение: ' . $data['message'],
            'old' => [
                'name' => '',
                'message' => '',
            ],
        ]);
    }
}


---

Что изменилось

Раньше было так:

if ($name === '') {
    $errors[] = 'Введите имя.';
}

if ($message === '') {
    $errors[] = 'Введите сообщение.';
}

А теперь так:

$validator = Validator::make($data)
    ->required('name', 'Введите имя.')
    ->min('name', 2, 'Имя должно быть не короче 2 символов.')
    ->required('message', 'Введите сообщение.')
    ->min('message', 5, 'Сообщение должно быть не короче 5 символов.');

То есть контроллер говорит:

Проверь name.
Проверь message.
Если есть ошибки — верни мне список.


---

3. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/form

Проверь так:

1. Отправь пустую форму.
2. Введи имя из 1 буквы.
3. Введи сообщение короче 5 символов.
4. Потом введи нормальные данные.

Должны появляться нормальные ошибки.


---

Что мы сделали простыми словами

Мы вынесли проверку данных из контроллера.

Было:

Controller сам проверяет каждое поле.

Стало:

Controller получает данные.
Validator проверяет данные.
Controller решает, что показать дальше.

То есть:

Controller — руководит.
Validator — проверяет.
View — показывает.


---

Почему это важно

В реальных проектах будет много форм:

создать сайт
создать страницу
создать заявку
создать папку
загрузить файл
назначить права

И везде надо проверять данные.

Теперь мы сможем использовать один общий инструмент:

$validator = Validator::make($data)
    ->required('title', 'Введите название.')
    ->max('title', 255, 'Название слишком длинное.');

Следующий шаг — сделать Session Flash-сообщения, чтобы после успешной POST-отправки делать редирект обратно и показывать сообщение, а не оставаться на /form/send.