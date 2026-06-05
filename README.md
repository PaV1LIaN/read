Идём дальше. Сейчас сделаем наш Route::resource() ещё ближе к Laravel.

Сейчас resource-маршруты у нас такие:

GET     /notes/{id:\d+}/edit
PUT     /notes/{id:\d+}
DELETE  /notes/{id:\d+}

А в Laravel обычно так:

GET     /notes/{note}/edit
PUT     /notes/{note}
DELETE  /notes/{note}

И контроллер:

public function edit(Note $note): Response

То есть название параметра совпадает с моделью:

{note} → Note $note

Сейчас сделаем так же.


---

1. Обнови resource() в /local/mvc/Core/Router.php

Найди метод:

public function resource(string $path, string $controller, array $options = []): void

И замени его полностью на:

public function resource(string $path, string $controller, array $options = []): void
{
    $path = '/' . trim($path, '/');

    $resourceName = trim($path, '/');
    $resourceName = str_replace('/', '.', $resourceName);

    /**
     * Laravel-like имя параметра.
     *
     * /notes  => {note}
     * /users  => {user}
     *
     * Можно переопределить:
     * Route::resource('/notes', NoteController::class, [
     *     'parameter' => 'note',
     * ]);
     */
    $parameter = (string)($options['parameter'] ?? $this->resourceParameterName($path));

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

    $modelParam = '{' . $parameter . ':\d+}';

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
        $this->get($path . '/' . $modelParam, [$controller, 'show'], $middleware, $resourceName . '.show');
    }

    if ($enabled('edit')) {
        $this->get($path . '/' . $modelParam . '/edit', [$controller, 'edit'], $middleware, $resourceName . '.edit');
    }

    if ($enabled('update')) {
        $this->put($path . '/' . $modelParam, [$controller, 'update'], $middleware, $resourceName . '.update');
    }

    if ($enabled('destroy')) {
        $this->delete($path . '/' . $modelParam, [$controller, 'destroy'], $middleware, $resourceName . '.destroy');
    }
}


---

2. В этот же Router.php добавь private-метод

Добавь внутрь класса Router, ближе к нижним private-методам:

private function resourceParameterName(string $path): string
{
    $path = trim($path, '/');

    if ($path === '') {
        return 'id';
    }

    $parts = explode('/', $path);
    $last = (string)end($parts);

    $last = str_replace('-', '_', $last);

    /**
     * Очень простое превращение множественного числа в единственное:
     * notes => note
     * users => user
     *
     * Для сложных слов можно использовать option:
     * 'parameter' => 'category'
     */
    if (str_ends_with($last, 'ies')) {
        return substr($last, 0, -3) . 'y';
    }

    if (str_ends_with($last, 's')) {
        return substr($last, 0, -1);
    }

    return $last;
}

Теперь:

Route::resource('/notes', NoteController::class)

создаст параметр:

{note:\d+}


---

3. Обнови Route Model Binding в /local/mvc/Core/Container.php

Найди в resolveParameters() кусок, где мы делали Model Binding:

if (is_subclass_of($className, Model::class) && array_key_exists('id', $parameters)) {
    $model = $className::findModel($parameters['id']);

    if (!$model instanceof Model) {
        throw new ModelNotFoundException($className, $parameters['id']);
    }

    $dependencies[] = $model;
    continue;
}

Замени на:

if (is_subclass_of($className, Model::class)) {
    /**
     * Laravel-like Route Model Binding.
     *
     * 1. Сначала ищем параметр по имени аргумента:
     *    public function edit(Note $note)
     *    маршрут: /notes/{note}
     *
     * 2. Потом fallback на старый вариант:
     *    /notes/{id}
     */
    $modelId = null;

    if (array_key_exists($name, $parameters)) {
        $modelId = $parameters[$name];
    } elseif (array_key_exists('id', $parameters)) {
        $modelId = $parameters['id'];
    } elseif (count($parameters) === 1) {
        $modelId = reset($parameters);
    }

    if ($modelId !== null) {
        $model = $className::findModel($modelId);

        if (!$model instanceof Model) {
            throw new ModelNotFoundException($className, $modelId);
        }

        $dependencies[] = $model;
        continue;
    }
}

Теперь контейнер понимает оба варианта:

{id}
{note}

Но Laravel-like вариант — это {note}.


---

4. Обнови ссылки в /local/mvc_demo/Views/notes/index.php

Теперь маршруты ждут параметр note, а не id.

Найди:

route('notes.edit', ['id' => (int)($note['id'] ?? 0)])

Замени на:

route('notes.edit', ['note' => (int)($note['id'] ?? 0)])

Найди:

route('notes.destroy', ['id' => (int)($note['id'] ?? 0)])

Замени на:

route('notes.destroy', ['note' => (int)($note['id'] ?? 0)])


---

5. Обнови ссылку в /local/mvc_demo/Views/notes/edit.php

Найди:

route('notes.update', ['id' => $noteId])

Замени на:

route('notes.update', ['note' => $noteId])


---

6. Проверь routes.php

Блок должен остаться таким:

Route::post('/notes/factory', [NoteController::class, 'factory'])
    ->middleware(['auth', 'admin', 'csrf'])
    ->name('notes.factory');

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

Важно: notes.factory должен быть выше Route::resource('/notes', ...).


---

7. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/debug/routes

Теперь маршруты должны стать такими:

GET     /notes                    notes.index
POST    /notes                    notes.store
GET     /notes/{note:\d+}/edit    notes.edit
PUT     /notes/{note:\d+}         notes.update
DELETE  /notes/{note:\d+}         notes.destroy

Потом проверь:

/local/mvc_demo/notes

Сценарии:

1. Изменить заметку.
2. Сохранить.
3. Удалить.
4. Открыть несуществующую заметку:
   /local/mvc_demo/notes/999999/edit

Для несуществующей заметки должен быть 404.


---

Что мы сделали

Раньше было просто:

/notes/{id}

Теперь Laravel-like:

/notes/{note}

И контроллер выглядит как в Laravel:

public function edit(Note $note): Response
{
    //
}

public function update(Note $note, UpdateNoteRequest $request): Response
{
    //
}

public function destroy(Note $note): Response
{
    //
}

Главная мысль:

{note} из URL превращается в объект Note.
Контроллер получает уже готовую модель.

Это прям очень близко к Laravel Route Model Binding.