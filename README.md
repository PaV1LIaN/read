Идём дальше. Сейчас сделаем Laravel-like pagination.

В Laravel обычно не делают так:

Note::latest(20)

А делают так:

Note::query()->paginate(10)

У нас QueryBuilder::paginate() уже есть, теперь подключим его к заметкам.


---

1. Обнови /local/mvc_demo/Models/Note.php

Добавь два метода внутрь класса Note:

public static function paginateLatest(int $page = 1, int $perPage = 10): array
{
    $result = self::query()
        ->orderBy('id', 'desc')
        ->paginate($page, $perPage);

    $result['items'] = array_map([self::class, 'normalize'], $result['items']);

    return $result;
}

public static function paginateTrashedLatest(int $page = 1, int $perPage = 10): array
{
    $result = self::onlyTrashed()
        ->orderBy('id', 'desc')
        ->paginate($page, $perPage);

    $result['items'] = array_map([self::class, 'normalize'], $result['items']);

    return $result;
}

То есть в модели теперь будут варианты:

Note::latest(20);              // просто последние 20
Note::paginateLatest($page);   // постранично
Note::trashedLatest(20);       // удалённые последние 20
Note::paginateTrashedLatest(); // удалённые постранично


---

2. Обнови /local/mvc_demo/Controllers/NoteController.php

Замени метод index():

public function index(): Response
{
    $this->authorize('viewAny', Note::class);

    return $this->render('notes/index', [
        'title' => 'Заметки',
        'notes' => Note::latest(20),
    ]);
}

на:

public function index(): Response
{
    $this->authorize('viewAny', Note::class);

    $page = (int)request('page', 1);

    $result = Note::paginateLatest($page, 10);

    return $this->render('notes/index', [
        'title' => 'Заметки',
        'notes' => $result['items'],
        'pagination' => $result['pagination'],
    ]);
}

Теперь замени метод trash():

public function trash(): Response
{
    $this->authorize('viewAny', Note::class);

    return $this->render('notes/trash', [
        'title' => 'Удалённые заметки',
        'notes' => Note::trashedLatest(20),
    ]);
}

на:

public function trash(): Response
{
    $this->authorize('viewAny', Note::class);

    $page = (int)request('page', 1);

    $result = Note::paginateTrashedLatest($page, 10);

    return $this->render('notes/trash', [
        'title' => 'Удалённые заметки',
        'notes' => $result['items'],
        'pagination' => $result['pagination'],
    ]);
}


---

3. Добавь helper для пагинации во view

Создай файл:

/local/mvc_demo/Views/partials/pagination.php

Код:

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

$pagination = $pagination ?? [];

$currentPage = (int)($pagination['page'] ?? $pagination['current_page'] ?? 1);
$lastPage = (int)($pagination['last_page'] ?? $pagination['pages'] ?? 1);
$total = (int)($pagination['total'] ?? 0);
$perPage = (int)($pagination['per_page'] ?? 10);

$routeName = (string)($routeName ?? '');
$routeParams = is_array($routeParams ?? null) ? $routeParams : [];
$query = is_array($query ?? null) ? $query : [];

if ($lastPage < 2 || $routeName === '') {
    return;
}

$from = $total > 0 ? (($currentPage - 1) * $perPage + 1) : 0;
$to = min($currentPage * $perPage, $total);

?>

<div class="mvc-info" style="display:flex;justify-content:space-between;gap:12px;align-items:center;flex-wrap:wrap;">
    <div style="color:#6b7280;">
        Показано <?= e($from) ?>–<?= e($to) ?> из <?= e($total) ?>
    </div>

    <div style="display:flex;gap:6px;align-items:center;flex-wrap:wrap;">
        <?php if ($currentPage > 1): ?>
            <a
                href="<?= e(route($routeName, $routeParams, array_merge($query, ['page' => $currentPage - 1]))) ?>"
                style="padding:6px 10px;border:1px solid #d1d5db;border-radius:8px;text-decoration:none;"
            >
                ← Назад
            </a>
        <?php endif; ?>

        <?php
        $start = max(1, $currentPage - 2);
        $end = min($lastPage, $currentPage + 2);
        ?>

        <?php for ($page = $start; $page <= $end; $page++): ?>
            <?php if ($page === $currentPage): ?>
                <span
                    style="padding:6px 10px;border-radius:8px;background:#2563eb;color:#fff;font-weight:600;"
                >
                    <?= e($page) ?>
                </span>
            <?php else: ?>
                <a
                    href="<?= e(route($routeName, $routeParams, array_merge($query, ['page' => $page]))) ?>"
                    style="padding:6px 10px;border:1px solid #d1d5db;border-radius:8px;text-decoration:none;"
                >
                    <?= e($page) ?>
                </a>
            <?php endif; ?>
        <?php endfor; ?>

        <?php if ($currentPage < $lastPage): ?>
            <a
                href="<?= e(route($routeName, $routeParams, array_merge($query, ['page' => $currentPage + 1]))) ?>"
                style="padding:6px 10px;border:1px solid #d1d5db;border-radius:8px;text-decoration:none;"
            >
                Вперёд →
            </a>
        <?php endif; ?>
    </div>
</div>


---

4. Подключи пагинацию в /local/mvc_demo/Views/notes/index.php

После таблицы заметок, ниже блока со списком, добавь:

<?php
$routeName = 'notes.index';
$routeParams = [];
$query = [];
require $_SERVER['DOCUMENT_ROOT'] . '/local/mvc_demo/Views/partials/pagination.php';
?>

То есть примерно после закрытия таблицы:

<?php endif; ?>
    </div>

    <?php
    $routeName = 'notes.index';
    $routeParams = [];
    $query = [];
    require $_SERVER['DOCUMENT_ROOT'] . '/local/mvc_demo/Views/partials/pagination.php';
    ?>
</div>


---

5. Подключи пагинацию в /local/mvc_demo/Views/notes/trash.php

Точно так же после таблицы удалённых заметок добавь:

<?php
$routeName = 'notes.trash';
$routeParams = [];
$query = [];
require $_SERVER['DOCUMENT_ROOT'] . '/local/mvc_demo/Views/partials/pagination.php';
?>


---

6. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/notes

Нажми несколько раз:

Создать 5 заметок через Factory

Когда заметок станет больше 10, внизу появится пагинация.

Проверь URL:

/local/mvc_demo/notes?page=2

И корзину:

/local/mvc_demo/notes/trash?page=2


---

Что мы сделали

Раньше список был простой:

Note::latest(20)

Теперь Laravel-like:

Note::paginateLatest($page, 10)

И внутри используется наш QueryBuilder:

$query->paginate($page, $perPage)

Главная мысль:

QueryBuilder собирает LIMIT/OFFSET.
Paginator считает страницы.
View показывает ссылки.

Это очень похоже на Laravel paginate(), только пока без Blade-компонента links().