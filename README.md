В этом файле есть подтверждённая проблема публичного доступа: сейчас страницы со статусом draft попадают в $pages, открываются напрямую через pageId, отображаются в навигации и могут оставаться в меню.

Исправляем это в /local/sitebuilder/lib/public_render.php.

1. Добавь проверку публичности страницы

Вставь этот код перед функцией sb_public_pages_for_site():

if (!function_exists('sb_public_page_is_visible')) {
    function sb_public_page_is_visible(array $page, array $pageMap): bool
    {
        $status = strtolower(trim((string)($page['status'] ?? 'draft')));

        if ($status !== 'published') {
            return false;
        }

        $currentPage = $page;
        $visited = [];

        while (true) {
            $currentId = (int)($currentPage['id'] ?? 0);

            if ($currentId <= 0 || isset($visited[$currentId])) {
                return false;
            }

            $visited[$currentId] = true;

            $parentId = (int)($currentPage['parentId'] ?? 0);

            if ($parentId <= 0) {
                return true;
            }

            if (!isset($pageMap[$parentId])) {
                return false;
            }

            $parentPage = $pageMap[$parentId];
            $parentStatus = strtolower(
                trim((string)($parentPage['status'] ?? 'draft'))
            );

            if ($parentStatus !== 'published') {
                return false;
            }

            $currentPage = $parentPage;
        }
    }
}

Эта функция:

скрывает draft;

скрывает страницу, если её родитель является draft;

скрывает страницу с несуществующим родителем;

защищает от циклической вложенности страниц.



---

2. Полностью замени sb_public_pages_for_site()

Было:

if (!function_exists('sb_public_pages_for_site')) {
    function sb_public_pages_for_site(int $siteId): array
    {
        $pages = array_values(array_filter(sb_read_pages(), static function ($p) use ($siteId) {
            return (int)($p['siteId'] ?? 0) === $siteId;
        }));

        usort($pages, static function ($a, $b) {
            $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);
            if ($sortCmp !== 0) {
                return $sortCmp;
            }
            return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
        });

        return array_map('sb_normalize_page_record', $pages);
    }
}

Должно стать:

if (!function_exists('sb_public_pages_for_site')) {
    function sb_public_pages_for_site(int $siteId): array
    {
        $pages = [];

        foreach (sb_read_pages() as $page) {
            if (!is_array($page)) {
                continue;
            }

            if ((int)($page['siteId'] ?? 0) !== $siteId) {
                continue;
            }

            $page = sb_normalize_page_record($page);
            $pageId = (int)($page['id'] ?? 0);

            if ($pageId <= 0) {
                continue;
            }

            $pages[$pageId] = $page;
        }

        $visiblePages = [];

        foreach ($pages as $page) {
            if (sb_public_page_is_visible($page, $pages)) {
                $visiblePages[] = $page;
            }
        }

        usort($visiblePages, static function ($a, $b) {
            $sortCmp =
                (int)($a['sort'] ?? 500)
                <=>
                (int)($b['sort'] ?? 500);

            if ($sortCmp !== 0) {
                return $sortCmp;
            }

            return
                (int)($a['id'] ?? 0)
                <=>
                (int)($b['id'] ?? 0);
        });

        return $visiblePages;
    }
}


---

3. Полностью замени sb_public_find_page_for_site()

Сейчас эта функция ищет страницу напрямую во всех страницах, поэтому может вернуть draft.

Замени её на:

if (!function_exists('sb_public_find_page_for_site')) {
    function sb_public_find_page_for_site(int $siteId, int $pageId): ?array
    {
        if ($siteId <= 0 || $pageId <= 0) {
            return null;
        }

        foreach (sb_public_pages_for_site($siteId) as $page) {
            if ((int)($page['id'] ?? 0) === $pageId) {
                return $page;
            }
        }

        return null;
    }
}

Теперь напрямую открыть черновую страницу через:

public.php?siteId=13&pageId=123

не получится.


---

4. Добавь фильтрацию пунктов меню

После функции sb_public_menu_for_site() вставь:

if (!function_exists('sb_public_filter_menu_pages')) {
    function sb_public_filter_menu_pages(?array $menu, array $pages): ?array
    {
        if (!$menu) {
            return null;
        }

        $allowedPageIds = [];

        foreach ($pages as $page) {
            $pageId = (int)($page['id'] ?? 0);

            if ($pageId > 0) {
                $allowedPageIds[$pageId] = true;
            }
        }

        $items = isset($menu['items']) && is_array($menu['items'])
            ? $menu['items']
            : [];

        $menu['items'] = array_values(array_filter(
            $items,
            static function ($item) use ($allowedPageIds) {
                if (!is_array($item)) {
                    return false;
                }

                $type = (string)($item['type'] ?? 'page');

                if ($type !== 'page') {
                    return true;
                }

                $pageId = (int)($item['pageId'] ?? 0);

                return $pageId > 0 && isset($allowedPageIds[$pageId]);
            }
        ));

        return $menu;
    }
}

Внешние ссылки останутся в меню, а пункты, связанные с черновыми или недоступными страницами, исчезнут.


---

5. Исправь выбор текущей страницы

В функции sb_public_build_view_model() найди:

$pages = sb_public_pages_for_site($siteId);
$currentPage = null;

if ($requestedPageId && $requestedPageId > 0) {
    $currentPage = sb_public_find_page_for_site($siteId, $requestedPageId);
}

if (!$currentPage) {
    $homePageId = (int)($site['homePageId'] ?? 0);
    if ($homePageId > 0) {
        $currentPage = sb_public_find_page_for_site($siteId, $homePageId);
    }
}

if (!$currentPage && !empty($pages)) {
    $currentPage = $pages[0];
}

Замени на:

$pages = sb_public_pages_for_site($siteId);
$currentPage = null;

$hasRequestedPage = $requestedPageId !== null && $requestedPageId > 0;

if ($hasRequestedPage) {
    $currentPage = sb_public_find_page_for_site(
        $siteId,
        (int)$requestedPageId
    );

    if (!$currentPage) {
        return null;
    }
}

if (!$hasRequestedPage) {
    $homePageId = (int)($site['homePageId'] ?? 0);

    if ($homePageId > 0) {
        $currentPage = sb_public_find_page_for_site(
            $siteId,
            $homePageId
        );
    }

    if (!$currentPage && !empty($pages)) {
        $currentPage = $pages[0];
    }
}

Раньше при запросе несуществующей или черновой страницы код молча открывал главную. Теперь такой запрос будет отклонён.


---

6. Исправь получение меню

Ниже в той же функции найди:

$layout = sb_public_layout_for_site($siteId);
$menu = sb_public_menu_for_site($site);

Замени на:

$layout = sb_public_layout_for_site($siteId);

$menu = sb_public_filter_menu_pages(
    sb_public_menu_for_site($site),
    $pages
);


---

7. Сделай сообщение в public.php точным

Сейчас при недоступной странице показывается:

<h1>Сайт не найден</h1>

Замени на:

<h1>Сайт или страница не найдены</h1>

После этих изменений публичная часть будет использовать только опубликованные страницы. Следующим нужно проверить шаблон:

/local/sitebuilder/views/layout/public_page.php

В нём находится фактический вывод меню, секций, боковой навигации и компонентов.