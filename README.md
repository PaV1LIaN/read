Отлично, идём дальше по цепочке.

Теперь падает уже на:

sb_normalize_page_record()

Значит у тебя в текущем наборе подключаемых lib нет нормализатора страницы.

Что сделать

Открой файл:

/local/sitebuilder/lib/helpers.php

И добавь туда этот блок:

if (!function_exists('sb_normalize_page_record')) {
    function sb_normalize_page_record(array $page): array
    {
        return [
            'id' => (int)($page['id'] ?? 0),
            'siteId' => (int)($page['siteId'] ?? 0),
            'title' => trim((string)($page['title'] ?? '')),
            'slug' => trim((string)($page['slug'] ?? '')),
            'parentId' => (int)($page['parentId'] ?? 0),
            'sort' => (int)($page['sort'] ?? 500),
            'status' => in_array((string)($page['status'] ?? 'draft'), ['draft', 'published'], true)
                ? (string)$page['status']
                : 'draft',
            'publishedAt' => !empty($page['publishedAt']) ? (string)$page['publishedAt'] : null,
            'createdAt' => !empty($page['createdAt']) ? (string)$page['createdAt'] : date('c'),
            'updatedAt' => !empty($page['updatedAt']) ? (string)$page['updatedAt'] : date('c'),
        ];
    }
}


---

Что ещё очень вероятно понадобится

Раз уж page.php уже использует page/block helpers, лучше сразу добавить ещё 2 функции, чтобы не ловить следующие fatals.

В тот же файл /local/sitebuilder/lib/helpers.php добавь сразу следом:

if (!function_exists('sb_normalize_block_record')) {
    function sb_normalize_block_record(array $block): array
    {
        return [
            'id' => (int)($block['id'] ?? 0),
            'pageId' => (int)($block['pageId'] ?? 0),
            'type' => trim((string)($block['type'] ?? 'text')),
            'sort' => (int)($block['sort'] ?? 500),
            'content' => is_array($block['content'] ?? null) ? $block['content'] : [],
            'props' => is_array($block['props'] ?? null) ? $block['props'] : [],
            'createdAt' => !empty($block['createdAt']) ? (string)$block['createdAt'] : date('c'),
            'updatedAt' => !empty($block['updatedAt']) ? (string)$block['updatedAt'] : date('c'),
        ];
    }
}

if (!function_exists('sb_normalize_menu_record')) {
    function sb_normalize_menu_record(array $menu): array
    {
        return [
            'id' => (int)($menu['id'] ?? 0),
            'siteId' => (int)($menu['siteId'] ?? 0),
            'name' => trim((string)($menu['name'] ?? '')),
            'items' => is_array($menu['items'] ?? null) ? array_values($menu['items']) : [],
            'createdAt' => !empty($menu['createdAt']) ? (string)$menu['createdAt'] : date('c'),
            'updatedAt' => !empty($menu['updatedAt']) ? (string)$menu['updatedAt'] : date('c'),
        ];
    }
}


---

Почему я советую добавить сразу всё это

Потому что у тебя сейчас backend постепенно упирается в отсутствующие базовые helper-функции.
Если добавить только sb_normalize_page_record, следующая ошибка может быть уже на:

sb_normalize_block_record()

sb_normalize_menu_record()

ещё каком-то общем helper-е


Поэтому лучше закрыть базу сразу.


---

Что делать сейчас

1.

Открой:

/local/sitebuilder/lib/helpers.php

2.

Добавь туда 3 функции:

sb_normalize_page_record

sb_normalize_block_record

sb_normalize_menu_record


3.

Сохрани файл

4.

Обнови editor.php

5.

Снова нажми Создать


---

Что ожидаю дальше

С большой вероятностью страница уже начнёт создаваться.
Если вылезет ещё одна ошибка, пришли следующий Response — добьём до конца.