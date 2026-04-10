Согласен, сейчас всё ещё не очень. Проблема уже не в цветах, а в самой модели меню.

Что сейчас выглядит плохо

слева дублируется корень: и заголовок Домашняя, и первый пункт Домашняя

дерево слишком “служебное”, а не навигационное

вертикальные линии и стрелки выглядят тяжеловато

активный пункт нормальный, но вся композиция слева всё ещё слабая

корень раздела лучше показывать как заголовок группы, а не как первый пункт списка


Как сделать лучше

Сделаем левое меню как чистую sidebar-навигацию:

сверху заголовок раздела

ниже дерево без дублирования корня

только дочерние страницы корня

у вложенных страниц — аккуратные отступы

toggle маленький и встроенный

без тяжёлых линий

активная страница выделена мягкой плашкой


Это уже будет выглядеть ближе к нормальному рабочему сайдбару.


---

Что поменять

Нужно изменить 2 места:

1. sb_public_render_section_nav() в lib/public_render.php


2. стили дерева в assets/public/public.css




---

1. Замени sb_public_render_section_nav() полностью

Файл:

/local/sitebuilder/lib/public_render.php

Замени функцию sb_public_render_section_nav на эту:

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

        $sortTree = function ($pageId) use (&$sortTree, &$pageMap) {
            if (!isset($pageMap[$pageId])) {
                return;
            }

            if (!empty($pageMap[$pageId]['children_nodes'])) {
                usort($pageMap[$pageId]['children_nodes'], function ($aId, $bId) use (&$pageMap) {
                    $a = $pageMap[$aId];
                    $b = $pageMap[$bId];

                    $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);
                    if ($sortCmp !== 0) {
                        return $sortCmp;
                    }

                    return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
                });

                foreach ($pageMap[$pageId]['children_nodes'] as $childId) {
                    $sortTree($childId);
                }
            }
        };

        $rootId = (int)($sectionRoot['id'] ?? 0);
        if (!isset($pageMap[$rootId])) {
            return '';
        }

        $sortTree($rootId);

        $isInActiveBranch = function ($nodeId) use (&$isInActiveBranch, &$pageMap, $currentPage) {
            if ((int)$nodeId === (int)($currentPage['id'] ?? 0)) {
                return true;
            }

            if (!isset($pageMap[$nodeId]) || empty($pageMap[$nodeId]['children_nodes'])) {
                return false;
            }

            foreach ($pageMap[$nodeId]['children_nodes'] as $childId) {
                if ($isInActiveBranch($childId)) {
                    return true;
                }
            }

            return false;
        };

        $renderNode = function ($nodeId, $depth) use (&$renderNode, &$pageMap, $currentPage, $basePath, $siteId, $isInActiveBranch) {
            if (!isset($pageMap[$nodeId])) {
                return '';
            }

            $node = $pageMap[$nodeId];
            $children = $node['children_nodes'] ?? [];
            $hasChildren = !empty($children);
            $isActive = (int)($node['id'] ?? 0) === (int)($currentPage['id'] ?? 0);
            $isOpen = $isInActiveBranch($nodeId);

            $activeClass = $isActive ? ' is-active' : '';
            $hasChildrenClass = $hasChildren ? ' has-children' : '';
            $openClass = $isOpen ? ' is-open' : '';

            $url = sb_public_h(sb_public_page_url($basePath, $siteId, (int)$node['id']));
            $title = sb_public_h((string)($node['title'] ?? 'Страница'));
            $depth = max(0, (int)$depth);

            $html = '';
            $html .= '<div class="sb-tree-node' . $hasChildrenClass . $openClass . '" style="--sb-nav-depth:' . $depth . ';">';
            $html .= '  <div class="sb-tree-node__row">';

            if ($hasChildren) {
                $html .= '    <button type="button" class="sb-tree-node__toggle" data-role="toggle" aria-expanded="' . ($isOpen ? 'true' : 'false') . '">';
                $html .= '      <span class="sb-tree-node__toggle-icon"></span>';
                $html .= '    </button>';
            } else {
                $html .= '    <span class="sb-tree-node__toggle sb-tree-node__toggle--empty"></span>';
            }

            $html .= '    <a class="sb-section-nav__link' . $activeClass . '" href="' . $url . '">';
            $html .= '      <span class="sb-section-nav__text">' . $title . '</span>';
            $html .= '    </a>';
            $html .= '  </div>';

            if ($hasChildren) {
                $html .= '  <div class="sb-tree-node__children">';
                foreach ($children as $childId) {
                    $html .= $renderNode($childId, $depth + 1);
                }
                $html .= '  </div>';
            }

            $html .= '</div>';

            return $html;
        };

        $html = '<div class="sb-section-nav">';
        $html .= '<div class="sb-section-nav__title-row">';
        $html .= '  <a class="sb-section-nav__root-link" href="' . sb_public_h(sb_public_page_url($basePath, $siteId, $rootId)) . '">'
              . sb_public_h((string)($sectionRoot['title'] ?? 'Раздел')) . '</a>';
        $html .= '</div>';

        $html .= '<div class="sb-section-nav__tree">';

        $rootChildren = $pageMap[$rootId]['children_nodes'] ?? [];
        foreach ($rootChildren as $childId) {
            $html .= $renderNode($childId, 0);
        }

        $html .= '</div>';
        $html .= '</div>';

        return $html;
    }
}


---

2. Замени стили sidebar-дерева

Файл:

/local/sitebuilder/assets/public/public.css

Найди все стили, связанные с:

.sb-section-nav

.sb-section-nav__title

.sb-section-nav__tree

.sb-tree-node

.sb-tree-node__row

.sb-tree-node__toggle

.sb-tree-node__toggle-icon

.sb-tree-node__children

.sb-section-nav__link

.sb-section-nav__text


И замени их на это:

.sb-section-nav {
    display: flex;
    flex-direction: column;
    gap: 12px;
}

.sb-section-nav__title-row {
    padding-bottom: 4px;
    border-bottom: 1px solid #f1f5f9;
}

.sb-section-nav__root-link {
    display: inline-flex;
    align-items: center;
    text-decoration: none;
    color: #111827;
    font-size: 14px;
    font-weight: 700;
}

.sb-section-nav__root-link:hover {
    color: var(--sb-accent);
}

.sb-section-nav__tree {
    display: flex;
    flex-direction: column;
    gap: 4px;
}

.sb-tree-node {
    display: flex;
    flex-direction: column;
    gap: 4px;
}

.sb-tree-node__row {
    display: flex;
    align-items: center;
    gap: 8px;
    padding-left: calc(var(--sb-nav-depth, 0) * 16px);
}

.sb-tree-node__toggle {
    width: 18px;
    min-width: 18px;
    height: 18px;
    border: 0;
    border-radius: 4px;
    background: transparent;
    cursor: pointer;
    position: relative;
    padding: 0;
    flex: 0 0 18px;
}

.sb-tree-node__toggle:hover {
    background: #f3f4f6;
}

.sb-tree-node__toggle--empty {
    cursor: default;
    pointer-events: none;
}

.sb-tree-node__toggle-icon {
    position: absolute;
    top: 50%;
    left: 50%;
    width: 6px;
    height: 6px;
    border-right: 2px solid #64748b;
    border-bottom: 2px solid #64748b;
    transform: translate(-50%, -60%) rotate(45deg);
    transition: transform .15s ease;
}

.sb-tree-node.is-open > .sb-tree-node__row .sb-tree-node__toggle-icon {
    transform: translate(-50%, -40%) rotate(225deg);
}

.sb-tree-node__children {
    display: none;
    flex-direction: column;
    gap: 4px;
}

.sb-tree-node.is-open > .sb-tree-node__children {
    display: flex;
}

.sb-section-nav__link {
    display: flex;
    align-items: center;
    min-height: 34px;
    width: 100%;
    text-decoration: none;
    color: #374151;
    padding: 6px 10px;
    border-radius: 8px;
    background: transparent;
    border: 1px solid transparent;
    transition: background .15s ease, color .15s ease, border-color .15s ease;
}

.sb-section-nav__link:hover {
    background: #f8fafc;
    border-color: #eef2f7;
}

.sb-section-nav__link.is-active {
    color: var(--sb-accent);
    background: #eef2ff;
    border-color: #c7d2fe;
    font-weight: 600;
}

.sb-section-nav__text {
    display: inline-block;
    font-size: 14px;
    line-height: 1.35;
}


---

Что изменится

После этого слева будет:

заголовок раздела: Домашняя

ниже уже только дочерние страницы, без повторного Домашняя

меню станет заметно чище

активная ветка останется

визуально будет ближе к нормальному sidebar, а не к техническому дереву



---

Почему это лучше

В твоём скриншоте главный визуальный мусор — это именно повторение корня:

хлебные крошки

h1 по центру

заголовок в sidebar

ещё раз корневой пункт внутри дерева


Сейчас мы убираем именно это дублирование.


---

Что проверить

На твоём примере ожидаемо должно стать так:

слева:

Домашняя как заголовок

Вложенная

Вложенная2

Вложенная3




А не:

Домашняя

Домашняя

Вложенная

...




---

Следующим шагом я бы ещё сделал левую колонку уже на 240–260px, а центральную шире, потому что сейчас контенту тесновато.