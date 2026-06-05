Скорее всего одна из двух причин:

1. Активных заметок меньше 11, поэтому пагинация специально не показывается.


2. В Paginator::toArray() у тебя другие ключи, и partial не понимает, сколько всего страниц.



Сделаем универсально.


---

1. Сначала проверь количество заметок

Нажми кнопку:

Создать 5 заметок через Factory

несколько раз, чтобы активных заметок было больше 10.

Потом открой:

/local/mvc_demo/notes?page=2

Если вторая страница открывается, значит пагинация работает, просто блок ссылок не отрисовывается.


---

2. Замени /local/mvc_demo/Views/partials/pagination.php

Полностью замени файл на этот вариант. Он понимает разные варианты ключей: page, current_page, pages, last_page, perPage, per_page.

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

$pagination = is_array($pagination ?? null) ? $pagination : [];

$routeName = (string)($routeName ?? '');
$routeParams = is_array($routeParams ?? null) ? $routeParams : [];
$query = is_array($query ?? null) ? $query : [];

$currentPage = (int)(
    $pagination['current_page']
    ?? $pagination['page']
    ?? 1
);

$lastPage = (int)(
    $pagination['last_page']
    ?? $pagination['pages']
    ?? $pagination['total_pages']
    ?? 1
);

$total = (int)(
    $pagination['total']
    ?? $pagination['items_total']
    ?? 0
);

$perPage = (int)(
    $pagination['per_page']
    ?? $pagination['perPage']
    ?? $pagination['limit']
    ?? 10
);

if ($currentPage < 1) {
    $currentPage = 1;
}

if ($lastPage < 1) {
    $lastPage = 1;
}

if ($perPage < 1) {
    $perPage = 10;
}

if ($routeName === '') {
    return;
}

/**
 * Если страниц меньше 2 — ссылки не нужны.
 */
if ($lastPage < 2) {
    return;
}

$from = $total > 0 ? (($currentPage - 1) * $perPage + 1) : 0;
$to = $total > 0 ? min($currentPage * $perPage, $total) : 0;

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

        <?php if ($start > 1): ?>
            <a
                href="<?= e(route($routeName, $routeParams, array_merge($query, ['page' => 1]))) ?>"
                style="padding:6px 10px;border:1px solid #d1d5db;border-radius:8px;text-decoration:none;"
            >
                1
            </a>

            <?php if ($start > 2): ?>
                <span style="padding:6px;color:#6b7280;">...</span>
            <?php endif; ?>
        <?php endif; ?>

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

        <?php if ($end < $lastPage): ?>
            <?php if ($end < $lastPage - 1): ?>
                <span style="padding:6px;color:#6b7280;">...</span>
            <?php endif; ?>

            <a
                href="<?= e(route($routeName, $routeParams, array_merge($query, ['page' => $lastPage]))) ?>"
                style="padding:6px 10px;border:1px solid #d1d5db;border-radius:8px;text-decoration:none;"
            >
                <?= e($lastPage) ?>
            </a>
        <?php endif; ?>

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

3. Если всё равно не видно — добавь временную диагностику

В /local/mvc_demo/Views/notes/index.php перед подключением pagination добавь:

<pre style="background:#111827;color:#fff;padding:12px;border-radius:10px;overflow:auto;">
<?= e(print_r($pagination ?? null, true)) ?>
</pre>

Вот сюда:

<?php
$routeName = 'notes.index';
$routeParams = [];
$query = [];

?>
<pre style="background:#111827;color:#fff;padding:12px;border-radius:10px;overflow:auto;">
<?= e(print_r($pagination ?? null, true)) ?>
</pre>
<?php
require $_SERVER['DOCUMENT_ROOT'] . '/local/mvc_demo/Views/partials/pagination.php';
?>

Потом открой:

/local/mvc_demo/notes

и пришли, что выводится в этом <pre>.


---

4. После проверки сбрось OPcache

opcache_reset();
echo 'OPcache reset OK';

Главное: если в pagination будет pages => 1 или last_page => 1, пагинация не появится, потому что страниц реально пока одна.