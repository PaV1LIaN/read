Да, вижу обе проблемы.

Что сейчас не так

1. По центру блок Разделы лишний

Ты прав — если слева уже есть section navigation, то в центре повторно выводить дочерние страницы не надо. Это визуальный мусор.

2. Слева дерево режется до 2 уровней по смыслу

Сейчас слева показывается:

корень раздела

его прямые дети


Но не показывается вся вложенная цепочка глубже.

3. Не видно, что один пункт вложен в другой

Ссылки слева сейчас плоские. Нет:

отступов по уровню

визуальной структуры дерева

понимания, что Вложенная2 лежит внутри Вложенная



---

Как правильно сделать

Нужно поправить только public_render.php.

Сделаем так:

уберём вывод child pages по центру

слева построим полноценное дерево раздела любой глубины

добавим уровни вложенности через отступы

активную страницу сохраним



---

Что заменить

Открой файл:

/local/sitebuilder/lib/public_render.php

И замени в нём только две функции:

1. sb_public_render_section_nav


2. sb_public_render_child_pages




---

1. Замени sb_public_render_section_nav(...)

Найди старую функцию sb_public_render_section_nav и замени её полностью на это:

if (!function_exists('sb_public_render_section_nav')) {
    function sb_public_render_section_nav(array $pages, ?array $currentPage, string $basePath, int $siteId): string
    {
        if (!$currentPage) {
            return '';
        }

        $sectionRoot = sb_public_section_root($pages, $currentPage);
        if (!$sectionRoot) {
            return '';
        }

        $pageMap = [];
        foreach ($pages as $page) {
            $page = sb_normalize_page_record($page);
            $page['children_nodes'] = [];
            $pageMap[(int)$page['id']] = $page;
        }

        foreach ($pageMap as $id => $page) {
            $parentId = (int)($page['parentId'] ?? 0);
            if ($parentId > 0 && isset($pageMap[$parentId])) {
                $pageMap[$parentId]['children_nodes'][] = $id;
            }
        }

        $sortChildren = function (&$node) use (&$pageMap, &$sortChildren) {
            if (!empty($node['children_nodes'])) {
                usort($node['children_nodes'], function ($aId, $bId) use (&$pageMap) {
                    $a = $pageMap[$aId];
                    $b = $pageMap[$bId];

                    $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);
                    if ($sortCmp !== 0) {
                        return $sortCmp;
                    }

                    return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
                });

                foreach ($node['children_nodes'] as $childId) {
                    $child = $pageMap[$childId];
                    $sortChildren($child);
                    $pageMap[$childId] = $child;
                }
            }
        };

        $rootId = (int)($sectionRoot['id'] ?? 0);
        if (!isset($pageMap[$rootId])) {
            return '';
        }

        $rootNode = $pageMap[$rootId];
        $sortChildren($rootNode);
        $pageMap[$rootId] = $rootNode;

        $renderNode = function ($pageId, $depth) use (&$renderNode, &$pageMap, $currentPage, $basePath, $siteId) {
            if (!isset($pageMap[$pageId])) {
                return '';
            }

            $node = $pageMap[$pageId];
            $active = (int)($node['id'] ?? 0) === (int)($currentPage['id'] ?? 0) ? ' is-active' : '';
            $url = sb_public_h(sb_public_page_url($basePath, $siteId, (int)$node['id']));
            $title = sb_public_h((string)($node['title'] ?? 'Страница'));
            $depth = max(0, (int)$depth);

            $html = '';
            $html .= '<a class="sb-section-nav__link' . $active . '" href="' . $url . '" style="--sb-nav-depth:' . $depth . ';">';
            $html .= '<span class="sb-section-nav__text">' . $title . '</span>';
            $html .= '</a>';

            if (!empty($node['children_nodes'])) {
                foreach ($node['children_nodes'] as $childId) {
                    $html .= $renderNode($childId, $depth + 1);
                }
            }

            return $html;
        };

        $html = '<div class="sb-section-nav">';
        $html .= '<div class="sb-section-nav__title">' . sb_public_h((string)($sectionRoot['title'] ?? 'Раздел')) . '</div>';
        $html .= '<div class="sb-section-nav__list">';
        $html .= $renderNode($rootId, 0);
        $html .= '</div>';
        $html .= '</div>';

        return $html;
    }
}


---

2. Замени sb_public_render_child_pages(...)

Найди функцию sb_public_render_child_pages и замени её полностью на это:

if (!function_exists('sb_public_render_child_pages')) {
    function sb_public_render_child_pages(array $pages, ?array $currentPage, string $basePath, int $siteId): string
    {
        return '';
    }
}

Это полностью уберёт блок Разделы из центра.


---

3. Добавь стили для вложенности слева

Теперь нужно немного подкрутить CSS.

Открой файл:

/local/sitebuilder/assets/public/public.css

Найди блок стилей для:

.sb-section-nav__title

.sb-section-nav__list

.sb-section-nav__link


И замени его полностью на это:

.sb-section-nav__title {
    font-size: 14px;
    font-weight: 700;
    margin-bottom: 12px;
    color: #111827;
}

.sb-section-nav__list {
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.sb-section-nav__link {
    display: block;
    text-decoration: none;
    color: #374151;
    padding: 10px 12px;
    padding-left: calc(12px + (var(--sb-nav-depth, 0) * 18px));
    border-radius: 10px;
    background: #f9fafb;
    border: 1px solid #e5e7eb;
    position: relative;
}

.sb-section-nav__link.is-active {
    color: var(--sb-accent);
    background: #eef2ff;
    border-color: #c7d2fe;
    font-weight: 600;
}

.sb-section-nav__link .sb-section-nav__text {
    display: inline-block;
}

.sb-section-nav__link[style*="--sb-nav-depth:1"]::before,
.sb-section-nav__link[style*="--sb-nav-depth:2"]::before,
.sb-section-nav__link[style*="--sb-nav-depth:3"]::before,
.sb-section-nav__link[style*="--sb-nav-depth:4"]::before,
.sb-section-nav__link[style*="--sb-nav-depth:5"]::before {
    content: "↳";
    position: absolute;
    left: calc(12px + ((var(--sb-nav-depth, 0) - 1) * 18px));
    top: 50%;
    transform: translateY(-50%);
    color: #94a3b8;
    font-size: 12px;
    font-weight: 700;
}


---

Что получится после этого

В центре

блок Разделы исчезнет


Слева

будет полноценное дерево:

корневая страница

дочерняя

дочерняя дочерней

и так дальше


И будет видно вложенность:

за счёт отступов

за счёт ↳



---

Что проверить

С твоим кейсом из 3 уровней:

например:

Домашняя

Вложенная

Вложенная2




Ожидаемый результат слева:

Домашняя

с отступом Вложенная

с ещё большим отступом Вложенная2


И активная страница должна выделяться.


---

Важный момент

Сейчас слева будет строиться дерево от корневого раздела, а не просто список прямых детей.
То есть проблема “показывает только 2” должна уйти.

Если после этого всё равно покажет не всю цепочку, значит проблема уже не в рендере, а в самих parentId у страниц. Тогда нужно будет посмотреть JSON страниц.