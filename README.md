Похоже, публичная страница вообще не увидела секции и вывела блоки старым способом — просто списком. Поэтому всё снова идёт в одну колонку.

Смотри по скрину: если бы секции реально отрендерились, в HTML был бы контейнер:

<div class="sb-section-grid" style="--sb-section-columns:3; ...">

А сейчас, скорее всего, его нет.

Нужно передать секции в $vm ещё на этапе сборки публичной страницы.


---

1. Исправь public_render.php

Файл:

/local/sitebuilder/lib/public_render.php

После функции:

function sb_public_page_blocks(int $pageId): array

добавь новую функцию:

if (!function_exists('sb_public_page_sections')) {
    function sb_public_page_sections(int $siteId, int $pageId): array
    {
        $sections = [];

        $repoFile = __DIR__ . '/PageSectionRepository.php';

        if (file_exists($repoFile)) {
            require_once $repoFile;
        }

        if (class_exists('PageSectionRepository')) {
            if (method_exists('PageSectionRepository', 'getByPage')) {
                $sections = PageSectionRepository::getByPage($siteId, $pageId);
            } elseif (method_exists('PageSectionRepository', 'listByPage')) {
                $sections = PageSectionRepository::listByPage($siteId, $pageId);
            } elseif (method_exists('PageSectionRepository', 'getList')) {
                $sections = PageSectionRepository::getList($siteId, $pageId);
            }
        }

        if (isset($sections['sections']) && is_array($sections['sections'])) {
            $sections = $sections['sections'];
        }

        if (!is_array($sections)) {
            $sections = [];
        }

        $sections = array_values(array_filter($sections, static function ($section) use ($siteId, $pageId) {
            return
                (int)($section['siteId'] ?? $section['site_id'] ?? 0) === $siteId &&
                (int)($section['pageId'] ?? $section['page_id'] ?? 0) === $pageId;
        }));

        usort($sections, static function ($a, $b) {
            $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);

            if ($sortCmp !== 0) {
                return $sortCmp;
            }

            return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
        });

        return $sections;
    }
}


---

2. В sb_public_build_view_model() добавь загрузку секций

В этой же функции найди:

$pageBlocks = $currentPage ? sb_public_page_blocks((int)$currentPage['id']) : [];

Сразу после добавь:

$pageSections = $currentPage
    ? sb_public_page_sections($siteId, (int)$currentPage['id'])
    : [];


---

3. В return добавь pageSections

Ниже в массиве return [ найди:

'pageBlocks' => $pageBlocks,

Сразу после добавь:

'pageSections' => $pageSections,

Должно быть так:

'pageBlocks' => $pageBlocks,
'pageSections' => $pageSections,
'layout' => $layout,


---

4. В public_page.php упрости получение секций

В файле:

/local/sitebuilder/views/layout/public_page.php

найди:

$pageSections = $currentPageId > 0
    ? sb_public_page_sections_for_page($vm, $siteId, $currentPageId)
    : [];

Замени на:

$pageSections = is_array($vm['pageSections'] ?? null) ? $vm['pageSections'] : [];


---

5. Проверь CSS

В public.css в конце должно быть именно это:

.sb-page-section > .sb-section-container > .sb-section-grid {
    display: grid;
    grid-template-columns: repeat(var(--sb-section-columns, 1), minmax(0, 1fr));
    gap: var(--sb-section-gap, 24px);
    width: 100%;
    min-width: 0;
    align-items: start;
    box-sizing: border-box;
}

.sb-page-section > .sb-section-container > .sb-section-grid > .sb-section-column {
    min-width: 0;
    box-sizing: border-box;
}


---

6. Как быстро проверить

После правок открой публичную страницу, нажми Ctrl + U и найди:

sb-section-grid

Если всё правильно, ты должен увидеть примерно:

<div class="sb-section-grid" style="--sb-section-columns:3;--sb-section-gap:24px;">

Если sb-section-grid нет — значит секции всё ещё не загружаются в публичную страницу.

Если sb-section-grid есть, но всё равно столбиком — значит проблема уже в CSS.