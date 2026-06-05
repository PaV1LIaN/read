Да, причина найдена.

У тебя pagination приходит так:

[total_pages] => 3
[pages] => Array
(
    [0] => 1
    [1] => 2
    [2] => 3
)

А в нашем partial было так:

$lastPage = (int)(
    $pagination['last_page']
    ?? $pagination['pages']
    ?? $pagination['total_pages']
    ?? 1
);

Из-за этого PHP брал pages, а это массив. Массив при (int) превращается в 1, поэтому partial думал:

lastPage = 1

и не показывал пагинацию.

Замени файл полностью.

/local/mvc_demo/Views/partials/pagination.php

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

/**
 * Важно:
 * pages у нас может быть массивом [1,2,3],
 * поэтому нельзя делать (int)$pagination['pages'].
 */
if (isset($pagination['last_page'])) {
    $lastPage = (int)$pagination['last_page'];
} elseif (isset($pagination['total_pages'])) {
    $lastPage = (int)$pagination['total_pages'];
} elseif (isset($pagination['pages']) && is_array($pagination['pages'])) {
    $lastPage = count($pagination['pages']);
} elseif (isset($pagination['pages'])) {
    $lastPage = (int)$pagination['pages'];
} else {
    $lastPage = 1;
}

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

if ($lastPage < 2) {
    return;
}

$from = (int)(
    $pagination['from']
    ?? ($total > 0 ? (($currentPage - 1) * $perPage + 1) : 0)
);

$to = (int)(
    $pagination['to']
    ?? ($total > 0 ? min($currentPage * $perPage, $total) : 0)
);

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

После этого убери временный debug-блок:

<pre>
<?= e(print_r($pagination ?? null, true)) ?>
</pre>

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

И открой:

/local/mvc_demo/notes

Теперь при твоих данных:

total = 26
page = 1
total_pages = 3

должны появиться страницы 1 2 3 и кнопка Вперёд →.