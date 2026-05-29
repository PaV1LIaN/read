Идём дальше по Laravel-подходу.

Сейчас у нас FormRequest уже есть, но контроллер всё ещё сам проверяет:

if ($request->fails()) {
    ...
}

А в Laravel обычно контроллеру уже приходит проверенный request. Если данные плохие — Laravel сам редиректит назад с ошибками.

Мы сделаем так же.

Хотим получить в FormController вот такой чистый код:

public function send(StoreMessageRequest $request): Response
{
    $data = $request->validated();

    Flash::success('Форма успешно отправлена. Имя: ' . $data['name']);

    return $this->redirectRoute('form.index');
}


---

1. Создай /local/mvc/Core/ValidationException.php

<?php

namespace Local\Mvc\Core;

use RuntimeException;

/**
 * ValidationException
 *
 * Исключение ошибки валидации.
 *
 * Если FormRequest не прошёл проверку,
 * он выбрасывает это исключение.
 */
class ValidationException extends RuntimeException
{
    public function __construct(
        private array $errors = [],
        private array $old = [],
        private string $redirectTo = ''
    ) {
        parent::__construct('Ошибка валидации');
    }

    public function errors(): array
    {
        return $this->errors;
    }

    public function old(): array
    {
        return $this->old;
    }

    public function redirectTo(): string
    {
        return $this->redirectTo;
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
}


---

2. Замени /local/mvc/Core/FormRequest.php

<?php

namespace Local\Mvc\Core;

/**
 * FormRequest
 *
 * Laravel-like request для валидации форм.
 */
abstract class FormRequest
{
    protected Request $request;

    private ?Validator $validator = null;

    public function __construct(Request $request)
    {
        $this->request = $request;
    }

    abstract public function rules(): array;

    public function messages(): array
    {
        return [];
    }

    public function authorize(): bool
    {
        return true;
    }

    /**
     * Имя маршрута, куда редиректить при ошибке.
     *
     * Если null — фреймворк попробует вернуть назад.
     */
    public function redirectRoute(): ?string
    {
        return null;
    }

    public function all(): array
    {
        return array_merge(
            $this->request->postAll(),
            $this->request->jsonAll()
        );
    }

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

    public function validated(): array
    {
        $data = $this->all();
        $validated = [];

        foreach (array_keys($this->rules()) as $field) {
            $value = $data[$field] ?? null;

            if (is_string($value)) {
                $value = trim($value);
            }

            $validated[$field] = $value;
        }

        return $validated;
    }

    /**
     * Автоматическая проверка.
     *
     * Container вызовет этот метод сам.
     */
    public function validateResolved(): void
    {
        if (!$this->fails()) {
            return;
        }

        $redirectTo = '';

        if ($this->redirectRoute()) {
            $redirectTo = App::route($this->redirectRoute());
        }

        throw new ValidationException(
            $this->errors(),
            $this->all(),
            $redirectTo
        );
    }
}


---

3. Обнови /local/mvc/Core/Container.php

Найди в методе resolveParameters() вот этот кусок:

if ($type instanceof ReflectionNamedType && !$type->isBuiltin()) {
    $dependencies[] = $this->make($type->getName());
    continue;
}

Замени на:

if ($type instanceof ReflectionNamedType && !$type->isBuiltin()) {
    $object = $this->make($type->getName());

    /**
     * Laravel-like поведение:
     * если в метод контроллера пришёл FormRequest,
     * валидируем его автоматически ДО запуска контроллера.
     */
    if ($object instanceof FormRequest) {
        $object->validateResolved();
    }

    $dependencies[] = $object;
    continue;
}

Теперь, если метод контроллера принимает StoreMessageRequest, контейнер сам его проверит.


---

4. Обнови /local/mvc/Core/ErrorHandler.php

Найди метод:

public static function renderThrowable(Throwable $e): void

В самое начало метода, сразу после открывающей {, добавь:

if ($e instanceof ValidationException) {
    self::renderValidationException($e);
    return;
}

Должно стать так:

public static function renderThrowable(Throwable $e): void
{
    if ($e instanceof ValidationException) {
        self::renderValidationException($e);
        return;
    }

    self::log($e->getMessage(), $e->getFile(), $e->getLine());

    ...
}

Теперь в этот же файл, перед методом debugEnabled(), добавь новые методы:

private static function renderValidationException(ValidationException $e): void
{
    self::log($e->getMessage(), $e->getFile(), $e->getLine());

    /**
     * Для API отдаём JSON, как в Laravel.
     */
    if (self::wantsJson()) {
        Response::json([
            'ok' => false,
            'error' => 'VALIDATION_ERROR',
            'details' => [
                'message' => 'Ошибка валидации',
                'errors' => $e->errors(),
            ],
        ], 422)->send();

        return;
    }

    /**
     * Для обычной формы:
     * 1. сохраняем старые значения
     * 2. сохраняем ошибки
     * 3. редиректим назад
     */
    Flash::old($e->old());

    foreach ($e->errorList() as $error) {
        Flash::error($error);
    }

    Response::redirect(self::validationRedirectUrl($e))->send();
}

private static function validationRedirectUrl(ValidationException $e): string
{
    if ($e->redirectTo() !== '') {
        return $e->redirectTo();
    }

    if (self::$request instanceof Request) {
        $referer = (string)self::$request->server('HTTP_REFERER', '');

        if (self::isSafeRedirectUrl($referer)) {
            return $referer;
        }
    }

    return App::projectUrl() . '/';
}

private static function isSafeRedirectUrl(string $url): bool
{
    $url = trim($url);

    if ($url === '') {
        return false;
    }

    if (str_starts_with($url, '/')) {
        return true;
    }

    if (!(self::$request instanceof Request)) {
        return false;
    }

    $currentHost = (string)self::$request->server('HTTP_HOST', '');

    $parts = parse_url($url);

    if (!is_array($parts)) {
        return false;
    }

    $urlHost = (string)($parts['host'] ?? '');

    if ($urlHost === '' || $currentHost === '') {
        return false;
    }

    return strcasecmp($urlHost, $currentHost) === 0;
}


---

5. Обнови /local/mvc_demo/Requests/StoreMessageRequest.php

Добавь метод redirectRoute():

public function redirectRoute(): ?string
{
    return 'form.index';
}

Полный файл:

<?php

namespace Local\MvcDemo\Requests;

use Local\Mvc\Core\FormRequest;

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

    public function redirectRoute(): ?string
    {
        return 'form.index';
    }
}


---

6. Упрости /local/mvc_demo/Controllers/FormController.php

Теперь контроллеру не нужно проверять ошибки.

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
        $data = $request->validated();

        Flash::success('Форма успешно отправлена. Имя: ' . $data['name']);

        return $this->redirectRoute('form.index');
    }
}

Вот теперь это уже прям Laravel-like.


---

7. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Проверь форму:

/local/mvc_demo/form

Сценарии:

1. Отправь пустую форму.
   Должны появиться ошибки, поля должны сохраниться.

2. Введи имя из 1 буквы и короткое сообщение.
   Должны появиться ошибки.

3. Введи нормальные данные.
   Должно появиться зелёное сообщение, поля очистятся.


---

Что мы сделали

Раньше контроллер сам проверял:

if ($request->fails()) {
    ...
}

Теперь контроллер получает уже валидный request:

public function send(StoreMessageRequest $request): Response
{
    $data = $request->validated();
}

Если данные плохие, контроллер вообще не запускается.

Цепочка теперь такая:

POST /form/send
  ↓
Router
  ↓
Container видит StoreMessageRequest
  ↓
FormRequest автоматически валидируется
  ↓
если ошибка — ValidationException
  ↓
ErrorHandler сохраняет Flash и редиректит обратно
  ↓
если всё хорошо — запускается FormController::send()

Это очень похоже на Laravel.