Да, это следующий правильный шаг.

Сейчас дерево уже есть, теперь добавим:

раскрытие/сворачивание дочерних страниц

авто-раскрытие активной ветки

нормальные toggles слева


Сделаем это без усложнения:

немного поменяем sb_public_render_section_nav() в public_render.php

добавим CSS в public.css

добавим маленький JS в public_page.php



---

Что поменять

1. Замени sb_public_render_section_nav() в lib/public_render.php

Открой файл:

/local/sitebuilder/lib/public_render.php

Найди функцию:

sb_public_render_section_nav(...)

И замени её полностью на эту:

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
            $html .= '<div class="sb-tree-node' . $hasChildrenClass . $openClass . '" data-node-id="' . (int)$node['id'] . '" style="--sb-nav-depth:' . $depth . ';">';
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
        $html .= '<div class="sb-section-nav__title">' . sb_public_h((string)($sectionRoot['title'] ?? 'Раздел')) . '</div>';
        $html .= '<div class="sb-section-nav__tree">';
        $html .= $renderNode($rootId, 0);
        $html .= '</div>';
        $html .= '</div>';

        return $html;
    }
}


---

2. Добавь стили в assets/public/public.css

В конец файла:

/local/sitebuilder/assets/public/public.css

добавь это:

.sb-section-nav__tree {
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.sb-tree-node {
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.sb-tree-node__row {
    display: flex;
    align-items: stretch;
    gap: 8px;
    padding-left: calc(var(--sb-nav-depth, 0) * 18px);
}

.sb-tree-node__toggle {
    width: 28px;
    min-width: 28px;
    height: 42px;
    border: 1px solid #e5e7eb;
    border-radius: 10px;
    background: #f9fafb;
    cursor: pointer;
    position: relative;
    padding: 0;
}

.sb-tree-node__toggle:hover {
    background: #f3f4f6;
}

.sb-tree-node__toggle--empty {
    border-color: transparent;
    background: transparent;
    cursor: default;
}

.sb-tree-node__toggle-icon {
    position: absolute;
    top: 50%;
    left: 50%;
    width: 8px;
    height: 8px;
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
    gap: 8px;
}

.sb-tree-node.is-open > .sb-tree-node__children {
    display: flex;
}

.sb-section-nav__link {
    display: flex;
    align-items: center;
    min-height: 42px;
    width: 100%;
    text-decoration: none;
    color: #374151;
    padding: 10px 12px;
    border-radius: 10px;
    background: #f9fafb;
    border: 1px solid #e5e7eb;
}

.sb-section-nav__link.is-active {
    color: var(--sb-accent);
    background: #eef2ff;
    border-color: #c7d2fe;
    font-weight: 600;
}

.sb-section-nav__text {
    display: inline-block;
}


---

3. Добавь JS в views/layout/public_page.php

Открой файл:

/local/sitebuilder/views/layout/public_page.php

И перед закрывающим </body> добавь этот скрипт:

<script>
document.addEventListener('click', function (e) {
    var toggle = e.target.closest('[data-role="toggle"]');
    if (!toggle) {
        return;
    }

    var node = toggle.closest('.sb-tree-node');
    if (!node) {
        return;
    }

    var isOpen = node.classList.contains('is-open');
    node.classList.toggle('is-open', !isOpen);
    toggle.setAttribute('aria-expanded', !isOpen ? 'true' : 'false');
});
</script>

То есть получится примерно так:

<?php if ($vm['showFooter']): ?>
        <footer class="sb-public-footer">
            <div class="sb-container">
                <?= $footerHtml !== '' ? $footerHtml : '<div class="sb-footer-note">© ' . date('Y') . ' ' . sb_public_h((string)($site['name'] ?? 'SiteBuilder')) . '</div>' ?>
            </div>
        </footer>
    <?php endif; ?>
</div>

<script>
document.addEventListener('click', function (e) {
    var toggle = e.target.closest('[data-role="toggle"]');
    if (!toggle) {
        return;
    }

    var node = toggle.closest('.sb-tree-node');
    if (!node) {
        return;
    }

    var isOpen = node.classList.contains('is-open');
    node.classList.toggle('is-open', !isOpen);
    toggle.setAttribute('aria-expanded', !isOpen ? 'true' : 'false');
});
</script>
</body>
</html>


---

Что получится

Теперь слева будет дерево такого вида:

Домашняя [toggle]

Вложенная [toggle]

Вложенная2

Вложенная3





И:

ветки можно раскрывать и закрывать

активная ветка будет открыта автоматически

остальные можно вручную свернуть



---

Что проверить

1. у страницы с детьми появилась кнопка-toggle


2. по нажатию дочерние элементы скрываются и показываются


3. активная ветка открыта при загрузке


4. глубокая вложенность видна корректно



Если хочешь, следующим сообщением можно сделать ещё аккуратнее: добавить запоминание раскрытых веток в localStorage, чтобы после перезагрузки страницы дерево оставалось в том же состоянии.