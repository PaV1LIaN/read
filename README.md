Да, идём дальше. Делаем FormRequest, как в Laravel.

Сейчас в FormController валидация выглядит так:

$validator = Validator::make($data)
    ->required('name', 'Введите имя.')
    ->min('name', 2, 'Имя должно быть не короче 2 символов.')
    ->max('name', 100, 'Имя должно быть не длиннее 100 символов.');

А в Laravel обычно делают отдельный request-класс:

public function send(StoreMessageRequest $request): Response
{
    $data = $request->validated();
}

То есть контроллер становится чище.


---

1. Обнови /local/mvc/Core/Validator.php

Полностью замени файл:

<?php

namespace Local\Mvc\Core;

/**
 * Validator
 *
 * Помощник для проверки данных.
 *
 * Поддерживает два стиля:
 *
 * 1. Старый fluent-стиль:
 * Validator::make($data)->required(...)->min(...)
 *
 * 2. Laravel-like rules:
 * Validator::validate($data, [
 *     'name' => ['required', 'min:2', 'max:100']
 * ]);
 */
class Validator
{
    private array $data;
    private array $errors = [];

    public function __construct(array $data)
    {
        $this->data = $data;
    }

    public static function make(array $data): self
    {
        return new self($data);
    }

    /**
     * Laravel-like валидация по правилам.
     */
    public static function validate(array $data, array $rules, array $messages = []): self
    {
        $validator = new self($data);

        foreach ($rules as $field => $fieldRules) {
            if (is_string($fieldRules)) {
                $fieldRules = explode('|', $fieldRules);
            }

            if (!is_array($fieldRules)) {
                continue;
            }

            foreach ($fieldRules as $rule) {
                $rule = trim((string)$rule);

                if ($rule === '') {
                    continue;
                }

                $validator->applyRule((string)$field, $rule, $messages);
            }
        }

        return $validator;
    }

    public function required(string $field, string $message): self
    {
        $value = $this->data[$field] ?? null;

        if ($value === null || trim((string)$value) === '') {
            $this->errors[$field][] = $message;
        }

        return $this;
    }

    public function min(string $field, int $length, string $message): self
    {
        $value = trim((string)($this->data[$field] ?? ''));

        if ($value !== '' && mb_strlen($value) < $length) {
            $this->errors[$field][] = $message;
        }

        return $this;
    }

    public function max(string $field, int $length, string $message): self
    {
        $value = trim((string)($this->data[$field] ?? ''));

        if ($value !== '' && mb_strlen($value) > $length) {
            $this->errors[$field][] = $message;
        }

        return $this;
    }

    public function fails(): bool
    {
        return !empty($this->errors);
    }

    public function passes(): bool
    {
        return !$this->fails();
    }

    public function errors(): array
    {
        return $this->errors;
    }

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

    private function applyRule(string $field, string $rule, array $messages): void
    {
        $value = $this->data[$field] ?? null;
        $valueString = trim((string)$value);

        [$ruleName, $ruleValue] = $this->parseRule($rule);

        if ($ruleName === 'required') {
            if ($value === null || $valueString === '') {
                $this->addError($field, $this->message($field, 'required', $messages, 'Поле обязательно для заполнения.'));
            }

            return;
        }

        if ($ruleName === 'min') {
            $min = (int)$ruleValue;

            if ($valueString !== '' && mb_strlen($valueString) < $min) {
                $this->addError($field, $this->message($field, 'min', $messages, 'Минимальная длина: ' . $min . '.'));
            }

            return;
        }

        if ($ruleName === 'max') {
            $max = (int)$ruleValue;

            if ($valueString !== '' && mb_strlen($valueString) > $max) {
                $this->addError($field, $this->message($field, 'max', $messages, 'Максимальная длина: ' . $max . '.'));
            }

            return;
        }

        if ($ruleName === 'integer') {
            if ($valueString !== '' && filter_var($valueString, FILTER_VALIDATE_INT) === false) {
                $this->addError($field, $this->message($field, 'integer', $messages, 'Поле должно быть целым числом.'));
            }

            return;
        }

        if ($ruleName === 'email') {
            if ($valueString !== '' && filter_var($valueString, FILTER_VALIDATE_EMAIL) === false) {
                $this->addError($field, $this->message($field, 'email', $messages, 'Некорректный email.'));
            }

            return;
        }
    }

    private function parseRule(string $rule): array
    {
        $parts = explode(':', $rule, 2);

        return [
            trim($parts[0]),
            trim($parts[1] ?? ''),
        ];
    }

    private function message(string $field, string $rule, array $messages, string $default): string
    {
        $key = $field . '.' . $rule;

        return (string)($messages[$key] ?? $default);
    }

    private function addError(string $field, string $message): void
    {
        $this->errors[$field][] = $message;
    }
}


---

2. Создай /local/mvc/Core/FormRequest.php

<?php

namespace Local\Mvc\Core;

/**
 * FormRequest
 *
 * Laravel-like request для валидации форм.
 *
 * Пример:
 *
 * class StoreMessageRequest extends FormRequest
 * {
 *     public function rules(): array
 *     {
 *         return [
 *             'name' => ['required', 'min:2']
 *         ];
 *     }
 * }
 */
abstract class FormRequest
{
    protected Request $request;

    private ?Validator $validator = null;

    public function __construct(Request $request)
    {
        $this->request = $request;
    }

    /**
     * Правила валидации.
     */
    abstract public function rules(): array;

    /**
     * Сообщения ошибок.
     */
    public function messages(): array
    {
        return [];
    }

    /**
     * Можно ли пользователю выполнять этот запрос.
     */
    public function authorize(): bool
    {
        return true;
    }

    /**
     * Все данные формы.
     */
    public function all(): array
    {
        return array_merge(
            $this->request->postAll(),
            $this->request->jsonAll()
        );
    }

    /**
     * Одно поле.
     */
    public function input(string $key, mixed $default = null): mixed
    {
        $data = $this->all();

        return $data[$key] ?? $default;
    }

    public function validator(): Validator
    {
        if ($this->validator instanceof Validator) {
            return $this->validator;
        }

        $this->validator = Validator::validate(
            $this->all(),
            $this->rules(),
            $this->messages()
        );

        return $this->validator;
    }

    public function fails(): bool
    {
        return !$this->authorize() || $this->validator()->fails();
    }

    public function errors(): array
    {
        if (!$this->authorize()) {
            return [
                'auth' => [
                    'Недостаточно прав для выполнения действия.',
                ],
            ];
        }

        return $this->validator()->errors();
    }

    public function errorList(): array
    {
        if (!$this->authorize()) {
            return [
                'Недостаточно прав для выполнения действия.',
            ];
        }

        return $this->validator()->errorList();
    }

    /**
     * Проверенные данные.
     *
     * Пока возвращаем только поля, которые есть в rules().
     */
    public function validated(): array
    {
        $data = $this->all();
        $validated = [];

        foreach (array_keys($this->rules()) as $field) {
            $validated[$field] = $data[$field] ?? null;
        }

        return $validated;
    }
}


---

3. Нужно добавить postAll() в Request.php

Открой:

/local/mvc/Core/Request.php

Добавь метод внутрь класса:

/**
 * Все POST-данные.
 */
public function postAll(): array
{
    return $this->post;
}

Если хочешь, рядом с методом:

public function post(string $key, mixed $default = null): mixed


---

4. Создай папку Requests в demo-проекте

/local/mvc_demo/Requests/


---

5. Создай /local/mvc_demo/Requests/StoreMessageRequest.php

<?php

namespace Local\MvcDemo\Requests;

use Local\Mvc\Core\FormRequest;

/**
 * StoreMessageRequest
 *
 * Проверка тестовой формы.
 *
 * Это очень похоже на Laravel FormRequest.
 */
class StoreMessageRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'name' => ['required', 'min:2', 'max:100'],
            'message' => ['required', 'min:5', 'max:1000'],
        ];
    }

    public function messages(): array
    {
        return [
            'name.required' => 'Введите имя.',
            'name.min' => 'Имя должно быть не короче 2 символов.',
            'name.max' => 'Имя должно быть не длиннее 100 символов.',

            'message.required' => 'Введите сообщение.',
            'message.min' => 'Сообщение должно быть не короче 5 символов.',
            'message.max' => 'Сообщение должно быть не длиннее 1000 символов.',
        ];
    }
}


---

6. Обнови /local/mvc_demo/Controllers/FormController.php

Полностью замени файл:

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Flash;
use Local\Mvc\Core\Response;
use Local\MvcDemo\Requests\StoreMessageRequest;

class FormController extends Controller
{
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

    public function send(StoreMessageRequest $request): Response
    {
        if ($request->fails()) {
            Flash::old($request->all());

            foreach ($request->errorList() as $error) {
                Flash::error($error);
            }

            return $this->redirectRoute('form.index');
        }

        $data = $request->validated();

        Flash::success('Форма успешно отправлена. Имя: ' . $data['name']);

        return $this->redirectRoute('form.index');
    }
}

Смотри, насколько стало похоже на Laravel:

public function send(StoreMessageRequest $request): Response
{
    $data = $request->validated();
}


---

7. Проверяем форму

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/form

Проверь:

1. Отправь пустую форму.
2. Введи имя из одной буквы.
3. Введи короткое сообщение.
4. Отправь нормальные данные.

Всё должно работать как раньше.


---

Что мы сделали

Раньше FormController сам знал правила:

Validator::make($data)
    ->required(...)
    ->min(...)

Теперь правила лежат отдельно:

/local/mvc_demo/Requests/StoreMessageRequest.php

Контроллер стал тоньше:

public function send(StoreMessageRequest $request): Response
{
    if ($request->fails()) {
        ...
    }

    $data = $request->validated();
}

Это уже очень похоже на Laravel.


---

Как это будет выглядеть в будущих проектах

Например, для sitebuilder:

public function store(StoreSiteRequest $request, SiteService $sites): Response
{
    $site = $sites->create($request->validated());

    return $this->redirectRoute('sites.edit', [
        'siteId' => $site['id'],
    ]);
}

А правила будут отдельно:

class StoreSiteRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'title' => ['required', 'min:3', 'max:255'],
            'code' => ['required', 'max:100'],
        ];
    }
}

Главная мысль:

Controller — принимает решение.
FormRequest — проверяет входные данные.
Service — выполняет бизнес-логику.
Model — работает с данными.

Это прям Laravel-подход.