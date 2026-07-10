В public_page.php подтверждены четыре проблемы:

1. Вложенные страницы в верхнем меню никогда не выводятся: $hasChildren всегда равен false.


2. Публичное редактирование таблиц обращается к старому /api.php.


3. Настроенное меню $menu загружается, но нигде не используется.


4. Хлебные крошки создаются в public_render.php, но шаблон их не выводит.



1. Исправь проверку дочерних активных страниц

Полностью замени функцию sb_public_auto_menu_has_active_child():

if (!function_exists('sb_public_auto_menu_has_active_child')) {
    function sb_public_auto_menu_has_active_child(
        array $pages,
        int $pageId,
        int $currentPageId,
        array $visited = []
    ): bool {
        if ($pageId <= 0 || isset($visited[$pageId])) {
            return false;
        }

        $visited[$pageId] = true;

        foreach ($pages as $page) {
            if ((int)($page['parentId'] ?? 0) !== $pageId) {
                continue;
            }

            $childId = (int)($page['id'] ?? 0);

            if ($childId <= 0 || isset($visited[$childId])) {
                continue;
            }

            if ($childId === $currentPageId) {
                return true;
            }

            if (
                sb_public_auto_menu_has_active_child(
                    $pages,
                    $childId,
                    $currentPageId,
                    $visited
                )
            ) {
                return true;
            }
        }

        return false;
    }
}

Здесь добавлена защита от циклической вложенности страниц.

2. Полностью замени построение автоматического меню

Замени функцию sb_public_render_auto_menu_level() целиком:

if (!function_exists('sb_public_render_auto_menu_level')) {
    function sb_public_render_auto_menu_level(
        array $pages,
        int $parentId,
        string $basePath,
        int $siteId,
        int $currentPageId,
        int $level = 0,
        array $visited = []
    ): string {
        if (isset($visited[$parentId])) {
            return '';
        }

        $visited[$parentId] = true;

        $children = sb_public_auto_menu_children(
            $pages,
            $parentId,
            $currentPageId
        );

        if (empty($children)) {
            return '';
        }

        $class = $level === 0
            ? 'sb-public-menu'
            : 'sb-public-menu__dropdown';

        $html = '<nav class="' . $class . '">';

        foreach ($children as $page) {
            $pageId = (int)($page['id'] ?? 0);

            if ($pageId <= 0 || isset($visited[$pageId])) {
                continue;
            }

            $title = (string)($page['title'] ?? 'Страница');
            $url = sb_public_page_url(
                $basePath,
                $siteId,
                $pageId
            );

            $childHtml = sb_public_render_auto_menu_level(
                $pages,
                $pageId,
                $basePath,
                $siteId,
                $currentPageId,
                $level + 1,
                $visited
            );

            $hasChildren = $childHtml !== '';

            $isActive =
                $pageId === $currentPageId
                || sb_public_auto_menu_has_active_child(
                    $pages,
                    $pageId,
                    $currentPageId
                );

            $html .= '<div class="sb-public-menu__item'
                . ($hasChildren ? ' has-children' : '')
                . ($isActive ? ' is-active' : '')
                . '">';

            $html .= '<a class="sb-public-menu__link" href="'
                . sb_public_h($url)
                . '">';

            $html .= sb_public_h($title);

            if ($hasChildren) {
                $html .= ' <span class="sb-public-menu__arrow">▾</span>';
            }

            $html .= '</a>';

            if ($hasChildren) {
                $html .= $childHtml;
            }

            $html .= '</div>';
        }

        $html .= '</nav>';

        return $html;
    }
}

В старой версии были строки:

$childHtml = '';
$hasChildren = false;

И эти значения больше нигде не изменялись. Поэтому вложенное меню физически не могло появиться.

3. Используй настроенное меню сайта

Сейчас находится строка:

$menuHtml = sb_public_render_auto_pages_menu($pages, $basePath, $siteId, (int)($currentPage['id'] ?? 0));

Замени её на:

$menuHtml = sb_public_render_menu(
    $menu,
    $basePath,
    $siteId
);

if ($menuHtml === '') {
    $menuHtml = sb_public_render_auto_pages_menu(
        $pages,
        $basePath,
        $siteId,
        (int)($currentPage['id'] ?? 0)
    );
}

Логика станет такой:

если для сайта назначено меню — используется оно;

если меню не назначено или пустое — автоматически строится меню из страниц.


Фильтрация черновых страниц в public_render.php из предыдущего шага должна быть уже внесена, иначе настроенное меню может содержать ссылки на draft.

4. Выведи хлебные крошки

Найди:

<main class="sb-public-main">
    <div class="sb-container">
        

        <div class="sb-layout

Замени на:

<main class="sb-public-main">
    <div class="sb-container">

        <?php if (!empty($vm['breadcrumbsHtml'])): ?>
            <div class="sb-public-breadcrumbs-row">
                <?= $vm['breadcrumbsHtml'] ?>
            </div>
        <?php endif; ?>

        <div class="sb-layout

Теперь данные, созданные здесь:

'breadcrumbsHtml' => sb_public_render_breadcrumbs(...)

будут реально отображаться.

5. Исправь API публичного редактирования

Внизу файла сейчас:

window.SB_PUBLIC_EDIT_CONFIG = <?= json_encode([
    'apiUrl' => $basePath . '/api.php',
    'sessid' => bitrix_sessid(),
], JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES) ?>;

Замени на:

window.SB_PUBLIC_EDIT_CONFIG = <?= json_encode([
    'apiUrl' => $basePath . '/api/index.php',
    'sessid' => bitrix_sessid(),
], JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES) ?>;

Это тот же рабочий endpoint, который уже подтверждён в редакторе:

/local/sitebuilder/api/index.php

6. Обнови версии ресурсов

Чтобы браузер не использовал старый кэш, измени:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=9">

на:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=10">

И для дискового компонента:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/components/disk/styles.css?v=5">

<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js?v=5"></script>

Проверка

После сохранения открой:

/local/sitebuilder/public.php?siteId=13

Проверь:

опубликованные страницы выводятся;

черновые страницы отсутствуют;

дочерние страницы раскрываются в верхнем меню;

активна текущая страница или её родительский раздел;

хлебные крошки появились над содержимым;

страница «Диск» продолжает загружать файлы;

адрес с &edit=Y больше не отправляет запросы на /api.php.