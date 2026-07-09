Идём дальше. Сейчас сделаем Laravel-like поиск + when() в QueryBuilder.

В Laravel часто пишут так:

Note::query()
    ->when($search, function ($query) use ($search) {
        $query->where('title', 'like', "%{$search}%");
    })
    ->paginate(10);

Сделаем похожее для наших заметок.


---

1. Обнови /local/mvc/Core/QueryBuilder.php

Внутрь класса QueryBuilder добавь метод, например рядом с where() / orWhere():

/**
 * Laravel-like when().
 *
 * Пример:
 * $query->when($search !== '', function ($query) use ($search) {
 *     $query->whereLike('title', $search);
 * });
 */
public function when(mixed $value, callable $callback, ?callable $default = null): self
{
    if ($value) {
        $callback($this, $value);
        return $this;
    }

    if ($default !== null) {
        $default($this, $value);
    }

    return $this;
}

Теперь можно будет условно добавлять фильтры.


---

2. Обнови /local/mvc_demo/Models/Note.php

Замени методы пагинации на эти:

public static function paginateLatest(int $page = 1, int $perPage = 10, string $search = ''): array
{
    $search = trim($search);

    $result = self::query()
        ->when($search !== '', function (\Local\Mvc\Core\QueryBuilder $query) use ($search) {
            $query->where(function (\Local\Mvc\Core\QueryBuilder $query) use ($search) {
                $query
                    ->whereRaw('CAST(id AS TEXT) LIKE :q', [
                        'q' => '%' . $search . '%',
                    ])
                    ->orWhereLike('title', $search)
                    ->orWhereLike('body', $search);
            });
        })
        ->orderBy('id', 'desc')
        ->paginate($page, $perPage);

    $result['items'] = array_map([self::class, 'normalize'], $result['items']);

    return $result;
}

public static function paginateTrashedLatest(int $page = 1, int $perPage = 10, string $search = ''): array
{
    $search = trim($search);

    $result = self::onlyTrashed()
        ->when($search !== '', function (\Local\Mvc\Core\QueryBuilder $query) use ($search) {
            $query->where(function (\Local\Mvc\Core\QueryBuilder $query) use ($search) {
                $query
                    ->whereRaw('CAST(id AS TEXT) LIKE :q', [
                        'q' => '%' . $search . '%',
                    ])
                    ->orWhereLike('title', $search)
                    ->orWhereLike('body', $search);
            });
        })
        ->orderBy('id', 'desc')
        ->paginate($page, $perPage);

    $result['items'] = array_map([self::class, 'normalize'], $result['items']);

    return $result;
}


---

3. Обнови NoteController

В /local/mvc_demo/Controllers/NoteController.php замени метод index() на:

public function index(): Response
{
    $this->authorize('viewAny', Note::class);

    $page = (int)request('page', 1);
    $search = trim((string)request('q', ''));

    $result = Note::paginateLatest($page, 10, $search);

    return $this->render('notes/index', [
        'title' => 'Заметки',
        'notes' => $result['items'],
        'pagination' => $result['pagination'],
        'search' => $search,
    ]);
}

И метод trash() на:

public function trash(): Response
{
    $this->authorize('viewAny', Note::class);

    $page = (int)request('page', 1);
    $search = trim((string)request('q', ''));

    $result = Note::paginateTrashedLatest($page, 10, $search);

    return $this->render('notes/trash', [
        'title' => 'Удалённые заметки',
        'notes' => $result['items'],
        'pagination' => $result['pagination'],
        'search' => $search,
    ]);
}


---

4. Добавь форму поиска в notes/index.php

В файле:

/local/mvc_demo/Views/notes/index.php

после блока со ссылкой на корзину добавь:

<div class="mvc-info">
    <form method="get" action="<?= e(route('notes.index')) ?>" style="display:flex;gap:10px;align-items:center;flex-wrap:wrap;">
        <input
            type="text"
            name="q"
            value="<?= e($search ?? '') ?>"
            placeholder="Поиск по ID, названию или тексту"
            style="flex:1;min-width:260px;min-height:42px;padding:8px 12px;border:1px solid #d1d5db;border-radius:10px;"
        >

        <button
            type="submit"
            style="min-height:42px;padding:0 18px;border:0;border-radius:10px;background:#2563eb;color:#fff;font-weight:600;cursor:pointer;"
        >
            Найти
        </button>

        <?php if (!empty($search)): ?>
            <a href="<?= e(route('notes.index')) ?>">
                Сбросить
            </a>
        <?php endif; ?>
    </form>
</div>

И внизу, где подключается пагинация, замени:

$query = [];

на:

$query = [];

if (!empty($search)) {
    $query['q'] = $search;
}

Должно получиться так:

<?php
$routeName = 'notes.index';
$routeParams = [];
$query = [];

if (!empty($search)) {
    $query['q'] = $search;
}

require $_SERVER['DOCUMENT_ROOT'] . '/local/mvc_demo/Views/partials/pagination.php';
?>


---

5. Добавь поиск в notes/trash.php

В файле:

/local/mvc_demo/Views/notes/trash.php

после ссылки назад добавь:

<div class="mvc-info">
    <form method="get" action="<?= e(route('notes.trash')) ?>" style="display:flex;gap:10px;align-items:center;flex-wrap:wrap;">
        <input
            type="text"
            name="q"
            value="<?= e($search ?? '') ?>"
            placeholder="Поиск по удалённым заметкам"
            style="flex:1;min-width:260px;min-height:42px;padding:8px 12px;border:1px solid #d1d5db;border-radius:10px;"
        >

        <button
            type="submit"
            style="min-height:42px;padding:0 18px;border:0;border-radius:10px;background:#2563eb;color:#fff;font-weight:600;cursor:pointer;"
        >
            Найти
        </button>

        <?php if (!empty($search)): ?>
            <a href="<?= e(route('notes.trash')) ?>">
                Сбросить
            </a>
        <?php endif; ?>
    </form>
</div>

И внизу перед подключением пагинации замени:

$query = [];

на:

$query = [];

if (!empty($search)) {
    $query['q'] = $search;
}


---

6. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Проверь:

/local/mvc_demo/notes?q=test
/local/mvc_demo/notes?page=2&q=test
/local/mvc_demo/notes/trash?q=test

Теперь поиск должен работать вместе с пагинацией: при переходе на страницу 2 параметр q не должен пропадать.


---

Что мы добавили:

$query->when($search !== '', function ($query) {
    ...
});

Это очень похоже на Laravel-подход: фильтры добавляются только если они нужны.