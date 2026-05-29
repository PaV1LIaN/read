Идём дальше. Сейчас сделаем Resource routes, как в Laravel.

В Laravel для CRUD обычно не пишут 5 маршрутов руками. Пишут примерно так:

Route::resource('notes', NoteController::class);

И Laravel сам создаёт маршруты:

GET     /notes              notes.index
POST    /notes              notes.store
GET     /notes/{id}/edit    notes.edit
PUT     /notes/{id}         notes.update
DELETE  /notes/{id}         notes.destroy

Мы сделаем похожее.


---

1. Обнови /local/mvc/Core/Router.php

Внутрь класса Router, рядом с методами get, post, put, delete, добавь метод:

public function resource(string $path, string $controller, array $options = []): void
{
    $path = '/' . trim($path, '/');

    $resourceName = trim($path, '/');
    $resourceName = str_replace('/', '.', $resourceName);

    $only = $options['only'] ?? [
        'index',
        'create',
        'store',
        'show',
        'edit',
        'update',
        'destroy',
    ];

    $except = $options['except'] ?? [];
    $middleware = $options['middleware'] ?? [];

    if (!is_array($only)) {
        $only = [$only];
    }

    if (!is_array($except)) {
        $except = [$except];
    }

    if (!is_array($middleware)) {
        $middleware = [$middleware];
    }

    $enabled = static function (string $action) use ($only, $except): bool {
        return in_array($action, $only, true) && !in_array($action, $except, true);
    };

    if ($enabled('index')) {
        $this->get($path, [$controller, 'index'], $middleware, $resourceName . '.index');
    }

    if ($enabled('create')) {
        $this->get($path . '/create', [$controller, 'create'], $middleware, $resourceName . '.create');
    }

    if ($enabled('store')) {
        $this->post($path, [$controller, 'store'], $middleware, $resourceName . '.store');
    }

    if ($enabled('show')) {
        $this->get($path . '/{id:\d+}', [$controller, 'show'], $middleware, $resourceName . '.show');
    }

    if ($enabled('edit')) {
        $this->get($path . '/{id:\d+}/edit', [$controller, 'edit'], $middleware, $resourceName . '.edit');
    }

    if ($enabled('update')) {
        $this->put($path . '/{id:\d+}', [$controller, 'update'], $middleware, $resourceName . '.update');
    }

    if ($enabled('destroy')) {
        $this->delete($path . '/{id:\d+}', [$controller, 'destroy'], $middleware, $resourceName . '.destroy');
    }
}


---

2. Обнови /local/mvc/Support/Facades/Route.php

Внутрь класса Route добавь метод:

public static function resource(string $path, string $controller, array $options = []): void
{
    self::router()->resource($path, $controller, $options);
}

Теперь можно будет писать:

Route::resource('/notes', NoteController::class);


---

3. Обнови /local/mvc_demo/Controllers/NoteController.php

У нас сейчас метод удаления называется:

delete()

В Laravel он называется:

destroy()

Полностью замени файл:

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
        return $this->render('notes/index', [
            'title' => 'Заметки',
            'notes' => Note::latest(20),
        ]);
    }

    public function store(StoreNoteRequest $request): Response
    {
        $data = $request->validated();

        Note::create([
            'title' => $data['title'],
            'body' => $data['body'],
            'created_at' => date('Y-m-d H:i:s'),
            'updated_at' => date('Y-m-d H:i:s'),
        ]);

        Flash::success('Заметка создана.');

        return redirect()->route('notes.index');
    }

    public function edit(string $id): Response
    {
        $note = Note::findNormalized((int)$id);

        if (!$note) {
            return Response::html(
                '<h1>404</h1><p>Заметка не найдена.</p>',
                404
            );
        }

        return $this->render('notes/edit', [
            'title' => 'Редактирование заметки',
            'note' => $note,
        ]);
    }

    public function update(string $id, UpdateNoteRequest $request): Response
    {
        $note = Note::findNormalized((int)$id);

        if (!$note) {
            return Response::html(
                '<h1>404</h1><p>Заметка не найдена.</p>',
                404
            );
        }

        $data = $request->validated();

        Note::updateById((int)$id, [
            'title' => $data['title'],
            'body' => $data['body'],
            'updated_at' => date('Y-m-d H:i:s'),
        ]);

        Flash::success('Заметка обновлена.');

        return redirect()->route('notes.index');
    }

    public function destroy(string $id): Response
    {
        Note::deleteById((int)$id);

        Flash::success('Заметка удалена.');

        return redirect()->route('notes.index');
    }
}


---

4. Обнови маршруты /local/mvc_demo/routes.php

Найди старые маршруты заметок:

Route::get('/notes', [NoteController::class, 'index'])
    ->name('notes.index');

Route::post('/notes', [NoteController::class, 'store'])
    ->middleware('csrf')
    ->name('notes.store');

Route::get('/notes/{id:\d+}/edit', [NoteController::class, 'edit'])
    ->name('notes.edit');

Route::put('/notes/{id:\d+}', [NoteController::class, 'update'])
    ->middleware('csrf')
    ->name('notes.update');

Route::delete('/notes/{id:\d+}', [NoteController::class, 'delete'])
    ->middleware('csrf')
    ->name('notes.delete');

Удали их и замени одним блоком:

Route::resource('/notes', NoteController::class, [
    'only' => [
        'index',
        'store',
        'edit',
        'update',
        'destroy',
    ],
    'middleware' => ['csrf'],
]);

Почему csrf можно повесить на все маршруты?

Потому что наш CsrfMiddleware проверяет только опасные методы:

POST
PUT
PATCH
DELETE

А GET он пропускает.


---

5. Обнови /local/mvc_demo/Views/notes/index.php

Найди маршрут удаления:

route('notes.delete', ['id' => (int)($note['id'] ?? 0)])

Замени на:

route('notes.destroy', ['id' => (int)($note['id'] ?? 0)])

То есть форма удаления должна быть такая:

<form method="post" action="<?= e(route('notes.destroy', ['id' => (int)($note['id'] ?? 0)])) ?>" style="margin:0;">
    <?= csrf_field() ?>
    <?= method_field('DELETE') ?>

    <button
        type="submit"
        onclick="return confirm('Удалить заметку?')"
        style="padding:6px 10px;border:0;border-radius:8px;background:#dc2626;color:#fff;cursor:pointer;"
    >
        Удалить
    </button>
</form>


---

6. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/debug/routes

Теперь должны быть маршруты:

GET     /notes              notes.index
POST    /notes              notes.store
GET     /notes/{id:\d+}/edit notes.edit
PUT     /notes/{id:\d+}      notes.update
DELETE  /notes/{id:\d+}      notes.destroy

Потом проверь:

/local/mvc_demo/notes

И сценарии:

создать заметку
изменить заметку
удалить заметку


---

Что мы сделали

Раньше CRUD-маршруты писались руками:

Route::get('/notes', ...);
Route::post('/notes', ...);
Route::get('/notes/{id}/edit', ...);
Route::put('/notes/{id}', ...);
Route::delete('/notes/{id}', ...);

Теперь Laravel-like:

Route::resource('/notes', NoteController::class, [
    'only' => ['index', 'store', 'edit', 'update', 'destroy'],
    'middleware' => ['csrf'],
]);

Главная мысль:

Resource route — это быстрый способ создать стандартные CRUD-маршруты.

Это ещё один шаг к Laravel.