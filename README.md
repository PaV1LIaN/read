Да, сейчас видно: публичная страница открывается, но $blocks приходит пустой. Значит проблема не в CSS и не в рендере блока, а в том, что views/layout/public_page.php не получает блоки из public.php.

Сделаем страховку: если $vm['blocks'] пустой, публичный шаблон сам попробует найти блоки по pageId.

В файле:

/local/sitebuilder/views/layout/public_page.php

1. Добавь функцию перед строкой $settings = ...

Найди ближе к низу файла:

$settings = is_array($site['settings'] ?? null) ? $site['settings'] : [];

Перед ней вставь:

if (!function_exists('sb_public_view_normalize_blocks')) {
    function sb_public_view_normalize_blocks(array $blocks): array
    {
        $result = [];

        foreach ($blocks as $block) {
            if (!is_array($block)) {
                continue;
            }

            $result[] = $block;
        }

        usort($result, static function ($a, $b) {
            $sortA = (int)($a['sort'] ?? $a['SORT'] ?? 500);
            $sortB = (int)($b['sort'] ?? $b['SORT'] ?? 500);

            if ($sortA !== $sortB) {
                return $sortA <=> $sortB;
            }

            return (int)($a['id'] ?? $a['ID'] ?? 0) <=> (int)($b['id'] ?? $b['ID'] ?? 0);
        });

        return $result;
    }
}

if (!function_exists('sb_public_view_filter_blocks_by_page')) {
    function sb_public_view_filter_blocks_by_page(array $blocks, int $pageId): array
    {
        if ($pageId <= 0) {
            return [];
        }

        $result = [];

        foreach ($blocks as $block) {
            if (!is_array($block)) {
                continue;
            }

            $blockPageId = (int)(
                $block['pageId']
                ?? $block['page_id']
                ?? $block['PAGE_ID']
                ?? 0
            );

            if ($blockPageId === $pageId) {
                $result[] = $block;
            }
        }

        return sb_public_view_normalize_blocks($result);
    }
}

if (!function_exists('sb_public_view_load_blocks_for_page')) {
    function sb_public_view_load_blocks_for_page(array $vm, array $currentPage, int $pageId): array
    {
        if (!empty($vm['blocks']) && is_array($vm['blocks'])) {
            return sb_public_view_normalize_blocks($vm['blocks']);
        }

        if (!empty($vm['pageBlocks']) && is_array($vm['pageBlocks'])) {
            return sb_public_view_normalize_blocks($vm['pageBlocks']);
        }

        if (!empty($vm['page_blocks']) && is_array($vm['page_blocks'])) {
            return sb_public_view_normalize_blocks($vm['page_blocks']);
        }

        if (!empty($currentPage['blocks']) && is_array($currentPage['blocks'])) {
            return sb_public_view_normalize_blocks($currentPage['blocks']);
        }

        if (!empty($currentPage['BLOCKS']) && is_array($currentPage['BLOCKS'])) {
            return sb_public_view_normalize_blocks($currentPage['BLOCKS']);
        }

        if (function_exists('sb_read_blocks')) {
            $allBlocks = sb_read_blocks();

            if (is_array($allBlocks)) {
                return sb_public_view_filter_blocks_by_page($allBlocks, $pageId);
            }
        }

        if (class_exists('BlockRepository')) {
            $methods = [
                'listByPageId',
                'getByPageId',
                'getListByPageId',
                'findByPageId',
                'listForPage',
            ];

            foreach ($methods as $method) {
                if (!method_exists('BlockRepository', $method)) {
                    continue;
                }

                try {
                    $loaded = BlockRepository::$method($pageId);

                    if (is_array($loaded)) {
                        return sb_public_view_normalize_blocks($loaded);
                    }
                } catch (Throwable $e) {
                    // Пробуем следующий способ.
                }
            }
        }

        return [];
    }
}

2. Замени определение $blocks

Сейчас вверху у тебя есть:

$blocks = is_array($vm['blocks'] ?? null) ? $vm['blocks'] : [];

Оставь как есть.

Но ниже, после строки:

$currentPageId = (int)($currentPage['id'] ?? 0);

сразу добавь:

$blocks = sb_public_view_load_blocks_for_page($vm, $currentPage, $currentPageId);

Должно получиться так:

$currentPageId = (int)($currentPage['id'] ?? 0);

$blocks = sb_public_view_load_blocks_for_page($vm, $currentPage, $currentPageId);

3. Временно добавь отладочный комментарий

Сразу после этой строки можно временно добавить:

echo "\n<!-- SB DEBUG: currentPageId={$currentPageId}; blocksCount=" . count($blocks) . " -->\n";

Потом открой исходный код страницы в браузере и найди SB DEBUG.

Если будет:

blocksCount=0

значит блоки не находятся по этому pageId.

Если будет:

blocksCount=1

значит блоки уже найдены, и дальше смотрим сам рендер.

Почему так произошло

Сейчас public.php передаёт в шаблон страницу, меню и сайт, но блоки в $vm['blocks'] либо не передаются, либо передаются под другим ключом. Поэтому публичный шаблон думал, что на странице блоков нет.

После этой вставки шаблон сам будет брать блоки из sb_read_blocks() или из BlockRepository, если такой класс есть.