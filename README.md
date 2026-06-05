Идём дальше. После Route Model Binding логичный Laravel-like шаг — Policies / Gate.

Простыми словами:

Middleware проверяет доступ ДО контроллера.
Policy проверяет: можно ли делать действие с конкретной моделью.

Например:

$this->authorize('update', $note);

Это значит:

Можно ли текущему пользователю редактировать именно эту заметку?


---

1. Создай /local/mvc/Core/AuthorizationException.php

<?php

namespace Local\Mvc\Core;

use RuntimeException;

class AuthorizationException extends RuntimeException
{
    public function __construct(
        private string $ability = '',
        private string $subject = ''
    ) {
        parent::__construct('Доступ запрещён');
    }

    public function ability(): string
    {
        return $this->ability;
    }

    public function subject(): string
    {
        return $this->subject;
    }
}


---

2. Создай /local/mvc/Core/GateManager.php

<?php

namespace Local\Mvc\Core;

use RuntimeException;

/**
 * GateManager
 *
 * Laravel-like Gate.
 *
 * Он ищет Policy-класс для модели и вызывает нужный метод.
 */
class GateManager
{
    public function allows(string $ability, object|string $subject): bool
    {
        $policy = $this->policyFor($subject);

        if (!$policy) {
            throw new RuntimeException('POLICY_NOT_FOUND_FOR_SUBJECT');
        }

        if (!method_exists($policy, $ability)) {
            throw new RuntimeException('POLICY_METHOD_NOT_FOUND: ' . get_class($policy) . '::' . $ability);
        }

        return (bool)$policy->{$ability}($subject);
    }

    public function denies(string $ability, object|string $subject): bool
    {
        return !$this->allows($ability, $subject);
    }

    public function authorize(string $ability, object|string $subject): void
    {
        if ($this->allows($ability, $subject)) {
            return;
        }

        $subjectName = is_object($subject) ? get_class($subject) : $subject;

        throw new AuthorizationException($ability, $subjectName);
    }

    private function policyFor(object|string $subject): ?object
    {
        $modelClass = is_object($subject) ? get_class($subject) : $subject;

        $policies = Config::get('auth.policies', []);

        if (!is_array($policies)) {
            return null;
        }

        $policyClass = (string)($policies[$modelClass] ?? '');

        if ($policyClass === '') {
            return null;
        }

        if (!class_exists($policyClass)) {
            throw new RuntimeException('POLICY_CLASS_NOT_FOUND: ' . $policyClass);
        }

        return App::make($policyClass);
    }
}


---

3. Создай facade /local/mvc/Support/Facades/Gate.php

<?php

namespace Local\Mvc\Support\Facades;

use Local\Mvc\Core\GateManager;

/**
 * Gate
 *
 * Laravel-like facade:
 *
 * Gate::allows('update', $note)
 * Gate::authorize('delete', $note)
 */
class Gate extends Facade
{
    protected static function accessor(): string
    {
        return GateManager::class;
    }
}


---

4. Зарегистрируй GateManager в /local/mvc/Core/App.php

В методе run() найди блок:

$container->singleton(\Local\Mvc\Core\SchemaBuilder::class, \Local\Mvc\Core\SchemaBuilder::class);
$container->singleton(\Local\Mvc\Core\SeederRunner::class, \Local\Mvc\Core\SeederRunner::class);

Добавь ниже:

$container->singleton(\Local\Mvc\Core\GateManager::class, \Local\Mvc\Core\GateManager::class);


---

5. Обнови /local/mvc/Core/Controller.php

Внутрь класса Controller добавь метод:

/**
 * Laravel-like authorize().
 *
 * Пример:
 * $this->authorize('update', $note);
 */
protected function authorize(string $ability, object|string $subject): void
{
    \Local\Mvc\Support\Facades\Gate::authorize($ability, $subject);
}

Теперь любой контроллер сможет писать:

$this->authorize('delete', $note);


---

6. Обнови /local/mvc/Core/ErrorHandler.php

В методе:

public static function renderThrowable(Throwable $e): void

в самое начало добавь:

if ($e instanceof AuthorizationException) {
    self::renderAuthorizationException($e);
    return;
}

Должно быть до ValidationException и до обычной ошибки.

Теперь в этот же класс добавь метод:

private static function renderAuthorizationException(AuthorizationException $e): void
{
    self::log($e->getMessage(), $e->getFile(), $e->getLine());

    if (self::wantsJson()) {
        Response::json([
            'ok' => false,
            'error' => 'FORBIDDEN',
            'details' => [
                'message' => 'Доступ запрещён.',
                'ability' => $e->ability(),
                'subject' => $e->subject(),
            ],
        ], 403)->send();

        return;
    }

    Response::html(
        '<h1>403</h1>'
        . '<p>Доступ запрещён.</p>'
        . '<pre>'
        . htmlspecialchars('Ability: ' . $e->ability() . "\nSubject: " . $e->subject())
        . '</pre>',
        403
    )->send();
}


---

7. Создай папку Policies

/local/mvc_demo/Policies/


---

8. Создай /local/mvc_demo/Policies/NotePolicy.php

<?php

namespace Local\MvcDemo\Policies;

use Local\Mvc\Core\Auth;
use Local\MvcDemo\Models\Note;

/**
 * NotePolicy
 *
 * Правила доступа к заметкам.
 *
 * Для demo:
 * - смотреть список может авторизованный пользователь
 * - создавать, редактировать и удалять может только админ
 */
class NotePolicy
{
    public function viewAny(string $modelClass): bool
    {
        return Auth::check();
    }

    public function create(string $modelClass): bool
    {
        return Auth::isAdmin();
    }

    public function update(Note $note): bool
    {
        return Auth::isAdmin();
    }

    public function delete(Note $note): bool
    {
        return Auth::isAdmin();
    }
}


---

9. Подключи Policy в /local/mvc_demo/config.php

Добавь блок auth:

'auth' => [
    'policies' => [
        \Local\MvcDemo\Models\Note::class => \Local\MvcDemo\Policies\NotePolicy::class,
    ],
],

Примерно так:

return [
    'app' => [
        'name' => 'MVC Demo',
        'description' => 'Тестовый проект на общем MVC-фреймворке',
    ],

    'debug' => true,

    'auth' => [
        'policies' => [
            \Local\MvcDemo\Models\Note::class => \Local\MvcDemo\Policies\NotePolicy::class,
        ],
    ],

    // остальные блоки ниже...
];


---

10. Обнови /local/mvc_demo/Controllers/NoteController.php

Теперь добавим проверки доступа.

Полный файл:

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Flash;
use Local\Mvc\Core\Response;
use Local\MvcDemo\Models\Note;
use Local\MvcDemo\Requests\StoreNoteRequest;
use Local\MvcDemo\Requests\UpdateNoteRequest;

class NoteController extends Controller
{
    public function index(): Response
    {
        $this->authorize('viewAny', Note::class);

        return $this->render('notes/index', [
            'title' => 'Заметки',
            'notes' => Note::latest(20),
        ]);
    }

    public function store(StoreNoteRequest $request): Response
    {
        $this->authorize('create', Note::class);

        $data = $request->validated();

        Note::create([
            'title' => $data['title'],
            'body' => $data['body'],
        ]);

        Flash::success('Заметка создана.');

        return redirect()->route('notes.index');
    }

    public function edit(Note $note): Response
    {
        $this->authorize('update', $note);

        return $this->render('notes/edit', [
            'title' => 'Редактирование заметки',
            'note' => $note->normalized(),
        ]);
    }

    public function update(Note $note, UpdateNoteRequest $request): Response
    {
        $this->authorize('update', $note);

        $data = $request->validated();

        $note->update([
            'title' => $data['title'],
            'body' => $data['body'],
        ]);

        Flash::success('Заметка обновлена.');

        return redirect()->route('notes.index');
    }

    public function destroy(Note $note): Response
    {
        $this->authorize('delete', $note);

        $note->delete();

        Flash::success('Заметка удалена.');

        return redirect()->route('notes.index');
    }

    public function factory(): Response
    {
        $this->authorize('create', Note::class);

        Note::factory()
            ->count(5)
            ->create();

        Flash::success('Factory создала 5 тестовых заметок.');

        return redirect()->route('notes.index');
    }
}


---

11. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Проверь:

/local/mvc_demo/notes

Если ты админ Битрикса — создание, изменение и удаление должны работать.

Если открыть под обычным пользователем:

создание / изменение / удаление

должны давать:

403 — Доступ запрещён


---

Что мы сделали

Теперь у нас есть Laravel-like авторизация:

$this->authorize('update', $note);

И Policy:

class NotePolicy
{
    public function update(Note $note): bool
    {
        return Auth::isAdmin();
    }
}

Главная мысль:

Middleware — общий входной фильтр.
Policy — проверка конкретного действия над конкретной моделью.

В Laravel это одна из самых важных идей.