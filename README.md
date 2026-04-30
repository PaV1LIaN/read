Да, давай сделаем проще: menu.php пока не удаляем, а просто перестаём использовать его в интерфейсе. Меню будет строиться автоматически из структуры страниц.

Логика будет такая:

Страницы сайта
├── Главная
├── Документы
│   ├── Приказы
│   └── Инструкции
└── Контакты

В публичной части получится меню:

Главная | Документы ▼ | Контакты
              ├ Приказы
              └ Инструкции

1. Убери ссылку на menu.php из editor.php

Найди в editor.php кнопку:

<a class="sb-btn sb-btn-light sb-btn-small" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/menu.php?siteId=<?= (int)$siteId ?>">Меню</a>

Удали её или закомментируй:

<?php /*
<a class="sb-btn sb-btn-light sb-btn-small" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/menu.php?siteId=<?= (int)$siteId ?>">Меню</a>
*/ ?>

menu.php останется в проекте, но пользователи не будут туда ходить.


---

2. В public_page.php замени формирование $menuHtml

Найди:

$menuHtml = sb_public_render_menu($menu, $basePath, $siteId);

Замени на:

$menuHtml = sb_public_render_auto_pages_menu($pages, $basePath, $siteId, (int)($currentPage['id'] ?? 0));


---

3. В начало public_page.php добавь функции автоменю

После блока переменных, например после:

$siteId = (int)$vm['siteId'];

добавь:

if (!function_exists('sb_public_auto_menu_is_page_visible')) {
    function sb_public_auto_menu_is_page_visible(array $page, int $currentPageId = 0): bool
    {
        $status = (string)($page['status'] ?? 'published');
        $pageId = (int)($page['id'] ?? 0);

        if ($pageId === $currentPageId) {
            return true;
        }

        return $status === 'published';
    }
}

if (!function_exists('sb_public_auto_menu_children')) {
    function sb_public_auto_menu_children(array $pages, int $parentId, int $currentPageId = 0): array
    {
        $items = [];

        foreach ($pages as $page) {
            if ((int)($page['parentId'] ?? 0) !== $parentId) {
                continue;
            }

            if (!sb_public_auto_menu_is_page_visible($page, $currentPageId)) {
                continue;
            }

            $items[] = $page;
        }

        usort($items, static function ($a, $b) {
            $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);

            if ($sortCmp !== 0) {
                return $sortCmp;
            }

            return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
        });

        return $items;
    }
}

if (!function_exists('sb_public_auto_menu_has_active_child')) {
    function sb_public_auto_menu_has_active_child(array $pages, int $pageId, int $currentPageId): bool
    {
        foreach ($pages as $page) {
            if ((int)($page['parentId'] ?? 0) !== $pageId) {
                continue;
            }

            $childId = (int)($page['id'] ?? 0);

            if ($childId === $currentPageId) {
                return true;
            }

            if (sb_public_auto_menu_has_active_child($pages, $childId, $currentPageId)) {
                return true;
            }
        }

        return false;
    }
}

if (!function_exists('sb_public_render_auto_menu_level')) {
    function sb_public_render_auto_menu_level(array $pages, int $parentId, string $basePath, int $siteId, int $currentPageId, int $level = 0): string
    {
        $children = sb_public_auto_menu_children($pages, $parentId, $currentPageId);

        if (empty($children)) {
            return '';
        }

        $class = $level === 0 ? 'sb-public-menu' : 'sb-public-menu__dropdown';

        $html = '<nav class="' . $class . '">';

        foreach ($children as $page) {
            $pageId = (int)($page['id'] ?? 0);
            $title = (string)($page['title'] ?? 'Страница');
            $url = sb_public_page_url($basePath, $siteId, $pageId);

            $childHtml = sb_public_render_auto_menu_level($pages, $pageId, $basePath, $siteId, $currentPageId, $level + 1);
            $hasChildren = $childHtml !== '';

            $isActive = $pageId === $currentPageId || sb_public_auto_menu_has_active_child($pages, $pageId, $currentPageId);

            $html .= '<div class="sb-public-menu__item' . ($hasChildren ? ' has-children' : '') . ($isActive ? ' is-active' : '') . '">';
            $html .= '<a class="sb-public-menu__link" href="' . sb_public_h($url) . '">';
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

if (!function_exists('sb_public_render_auto_pages_menu')) {
    function sb_public_render_auto_pages_menu(array $pages, string $basePath, int $siteId, int $currentPageId = 0): string
    {
        return sb_public_render_auto_menu_level($pages, 0, $basePath, $siteId, $currentPageId, 0);
    }
}


---

4. Убери старый fallback в header

В public_page.php найди блок:

<?php if ($menuHtml !== ''): ?>
    <?= $menuHtml ?>
<?php elseif (!empty($pages)): ?>
    <nav class="sb-public-menu">
        <?php foreach ($pages as $page): ?>
            <?php if ((int)($page['parentId'] ?? 0) !== 0) continue; ?>
            <a class="sb-public-menu__link" href="<?= sb_public_h(sb_public_page_url($basePath, $siteId, (int)$page['id'])) ?>">
                <?= sb_public_h((string)($page['title'] ?? 'Страница')) ?>
            </a>
        <?php endforeach; ?>
    </nav>
<?php endif; ?>

Замени на:

<?php if ($menuHtml !== ''): ?>
    <?= $menuHtml ?>
<?php endif; ?>

Теперь меню будет только из структуры страниц.


---

5. Добавь CSS в public.css

Файл:

/local/sitebuilder/assets/public/public.css

Добавь в конец:

.sb-public-menu {
    display: flex;
    align-items: center;
    gap: 6px;
    flex-wrap: wrap;
}

.sb-public-menu__item {
    position: relative;
}

.sb-public-menu__link {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    min-height: 36px;
    padding: 0 12px;
    border-radius: 10px;
    color: #111827;
    text-decoration: none;
    font-size: 14px;
    font-weight: 600;
    transition: background .15s ease, color .15s ease;
}

.sb-public-menu__link:hover,
.sb-public-menu__item.is-active > .sb-public-menu__link {
    background: rgba(37, 99, 235, 0.08);
    color: var(--sb-accent, #2563eb);
}

.sb-public-menu__arrow {
    font-size: 11px;
    opacity: .7;
}

.sb-public-menu__dropdown {
    position: absolute;
    left: 0;
    top: calc(100% + 6px);
    z-index: 50;
    min-width: 220px;
    display: none;
    flex-direction: column;
    gap: 2px;
    padding: 8px;
    border: 1px solid #e5e7eb;
    border-radius: 14px;
    background: #fff;
    box-shadow: 0 18px 45px rgba(15, 23, 42, .14);
}

.sb-public-menu__dropdown .sb-public-menu__item {
    width: 100%;
}

.sb-public-menu__dropdown .sb-public-menu__link {
    width: 100%;
    justify-content: space-between;
    min-height: 34px;
    padding: 0 10px;
    border-radius: 9px;
    white-space: nowrap;
}

.sb-public-menu__item:hover > .sb-public-menu__dropdown {
    display: flex;
}

.sb-public-menu__dropdown .sb-public-menu__dropdown {
    left: calc(100% + 8px);
    top: 0;
}

@media (max-width: 760px) {
    .sb-public-menu {
        align-items: stretch;
        width: 100%;
        flex-direction: column;
        gap: 4px;
    }

    .sb-public-menu__item {
        width: 100%;
    }

    .sb-public-menu__link {
        width: 100%;
        justify-content: space-between;
    }

    .sb-public-menu__dropdown {
        position: static;
        display: flex;
        box-shadow: none;
        border-radius: 12px;
        margin: 4px 0 4px 12px;
        min-width: 0;
    }

    .sb-public-menu__dropdown .sb-public-menu__dropdown {
        position: static;
        margin-left: 12px;
    }
}


---

После этого menu.php можно считать техническим файлом “на потом”, а реальное меню будет строиться автоматически из страниц. Следующий шаг — если всё заведётся, уберём кнопку Меню также из layout.php/других мест, где она может встречаться.