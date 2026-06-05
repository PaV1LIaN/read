Да, это логичный баг: сейчас шаблон сохраняет страницы, блоки, layout, меню, но секции страницы отдельно не сохраняет. Поэтому при создании сайта из шаблона создаётся дефолтная секция на 1 колонку, и блоки съезжают туда.

Правим один файл:

/local/sitebuilder/lib/SiteTemplateService.php


---

1. В createFromSite() добавь сохранение секций

Найди внутри функции createFromSite():

$menus = self::menusForSite($siteId);

Замени на:

$sections = self::sectionsForSite($siteId, $pages);
$menus = self::menusForSite($siteId);

Ниже найди payload:

'payload' => [
    'site' => self::prepareSiteForSnapshot($site),
    'pages' => array_map([self::class, 'preparePageForSnapshot'], $pages),
    'blocks' => $blocks,
    'layout' => $layout,
    'menus' => array_map([self::class, 'prepareMenuForSnapshot'], $menus),
],

Замени на:

'payload' => [
    'site' => self::prepareSiteForSnapshot($site),
    'pages' => array_map([self::class, 'preparePageForSnapshot'], $pages),
    'sections' => array_map([self::class, 'prepareSectionForSnapshot'], $sections),
    'blocks' => $blocks,
    'layout' => $layout,
    'menus' => array_map([self::class, 'prepareMenuForSnapshot'], $menus),
],


---

2. В createSiteFromTemplate() добавь копирование секций

Найди:

$pageIdMap = self::copyPages($siteId, $payload, $userId);
self::copyBlocks($pageIdMap, $payload, $userId);

Замени на:

$pageIdMap = self::copyPages($siteId, $payload, $userId);
$sectionIdMap = self::copySections($siteId, $pageIdMap, $payload, $userId);
self::copyBlocks($pageIdMap, $payload, $userId, $sectionIdMap);


---

3. Замени функцию prepareBlockForSnapshot()

Найди:

protected static function prepareBlockForSnapshot(array $block): array

И замени всю функцию на эту:

protected static function prepareBlockForSnapshot(array $block): array
{
    $rawProps = is_array($block['props'] ?? null) ? $block['props'] : [];
    $placement = is_array($rawProps['_placement'] ?? null) ? $rawProps['_placement'] : [];

    $sectionId = (int)($block['sectionId'] ?? 0);

    if ($sectionId <= 0) {
        $sectionId = (int)($rawProps['sectionId'] ?? 0);
    }

    if ($sectionId <= 0) {
        $sectionId = (int)($placement['sectionId'] ?? 0);
    }

    $column = (int)($block['column'] ?? 0);

    if ($column <= 0) {
        $column = (int)($rawProps['column'] ?? 0);
    }

    if ($column <= 0) {
        $column = (int)($placement['column'] ?? 0);
    }

    if ($column <= 0) {
        $column = 1;
    }

    $block = sb_normalize_block_record($block);

    return [
        'oldId' => (int)($block['id'] ?? 0),
        'oldPageId' => (int)($block['pageId'] ?? 0),
        'oldSectionId' => $sectionId,
        'sectionId' => $sectionId,
        'column' => max(1, min(4, $column)),
        'type' => (string)($block['type'] ?? 'text'),
        'sort' => (int)($block['sort'] ?? 500),
        'content' => self::sanitizeDiskData($block['content'] ?? []),
        'props' => self::sanitizeDiskData($block['props'] ?? []),
    ];
}


---

4. Добавь новые функции перед menusForSite()

Найди функцию:

protected static function menusForSite(int $siteId): array

Прямо перед ней вставь:

protected static function sectionsForSite(int $siteId, array $pages): array
{
    $pageIds = [];

    foreach ($pages as $page) {
        $pageId = (int)($page['id'] ?? 0);

        if ($pageId > 0) {
            $pageIds[$pageId] = true;
        }
    }

    if (empty($pageIds)) {
        return [];
    }

    $repoFile = __DIR__ . '/PageSectionRepository.php';

    if (file_exists($repoFile)) {
        require_once $repoFile;
    }

    if (!class_exists('PageSectionRepository')) {
        return [];
    }

    $sections = array_values(array_filter(PageSectionRepository::readAll(), static function ($section) use ($siteId, $pageIds) {
        $sectionSiteId = (int)($section['siteId'] ?? 0);
        $sectionPageId = (int)($section['pageId'] ?? 0);

        return $sectionSiteId === $siteId && isset($pageIds[$sectionPageId]);
    }));

    usort($sections, static function ($a, $b) {
        $pageCmp = (int)($a['pageId'] ?? 0) <=> (int)($b['pageId'] ?? 0);

        if ($pageCmp !== 0) {
            return $pageCmp;
        }

        $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);

        if ($sortCmp !== 0) {
            return $sortCmp;
        }

        return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
    });

    return $sections;
}

protected static function prepareSectionForSnapshot(array $section): array
{
    return [
        'oldId' => (int)($section['id'] ?? 0),
        'oldPageId' => (int)($section['pageId'] ?? 0),
        'title' => (string)($section['title'] ?? 'Секция'),
        'sort' => (int)($section['sort'] ?? 500),
        'layout' => is_array($section['layout'] ?? null) ? $section['layout'] : [],
        'props' => is_array($section['props'] ?? null) ? $section['props'] : [],
    ];
}

protected static function copySections(int $siteId, array $pageIdMap, array $payload, int $userId): array
{
    $templateSections = is_array($payload['sections'] ?? null) ? $payload['sections'] : [];

    if (empty($templateSections)) {
        return [];
    }

    $repoFile = __DIR__ . '/PageSectionRepository.php';

    if (file_exists($repoFile)) {
        require_once $repoFile;
    }

    if (!class_exists('PageSectionRepository')) {
        return [];
    }

    $items = PageSectionRepository::readAll();
    $nextSectionId = self::nextSectionId($items);
    $now = date('c');

    $sectionIdMap = [];
    $newSections = [];

    foreach ($templateSections as $section) {
        $oldPageId = (int)($section['oldPageId'] ?? 0);

        if ($oldPageId <= 0 || !isset($pageIdMap[$oldPageId])) {
            continue;
        }

        $oldSectionId = (int)($section['oldId'] ?? 0);
        $newSectionId = $nextSectionId++;

        if ($oldSectionId > 0) {
            $sectionIdMap[$oldSectionId] = $newSectionId;
        }

        $newSections[] = [
            'id' => $newSectionId,
            'siteId' => $siteId,
            'pageId' => (int)$pageIdMap[$oldPageId],
            'type' => 'section',
            'title' => (string)($section['title'] ?? 'Секция'),
            'sort' => (int)($section['sort'] ?? 500),
            'layout' => is_array($section['layout'] ?? null) ? $section['layout'] : [],
            'props' => is_array($section['props'] ?? null) ? $section['props'] : [],
            'createdBy' => $userId,
            'createdAt' => $now,
            'updatedBy' => $userId,
            'updatedAt' => $now,
        ];
    }

    if (!empty($newSections)) {
        PageSectionRepository::writeAll(array_merge($items, $newSections));
    }

    return $sectionIdMap;
}

protected static function nextSectionId(array $sections): int
{
    $maxId = 0;

    foreach ($sections as $section) {
        $maxId = max($maxId, (int)($section['id'] ?? 0));
    }

    return $maxId + 1;
}


---

5. Замени функцию copyBlocks()

Найди:

protected static function copyBlocks(array $pageIdMap, array $payload, int $userId): void

И замени всю функцию на эту:

protected static function copyBlocks(array $pageIdMap, array $payload, int $userId, array $sectionIdMap = []): void
{
    $blocks = sb_read_blocks();
    $templateBlocks = is_array($payload['blocks'] ?? null) ? $payload['blocks'] : [];
    $nextBlockId = sb_next_block_id($blocks);
    $now = date('c');

    foreach ($templateBlocks as $block) {
        $oldPageId = (int)($block['oldPageId'] ?? 0);

        if (!isset($pageIdMap[$oldPageId])) {
            continue;
        }

        $props = self::sanitizeDiskData($block['props'] ?? []);

        if (!is_array($props)) {
            $props = [];
        }

        $placement = is_array($props['_placement'] ?? null) ? $props['_placement'] : [];

        $oldSectionId = (int)($block['oldSectionId'] ?? $block['sectionId'] ?? 0);

        if ($oldSectionId <= 0) {
            $oldSectionId = (int)($props['sectionId'] ?? 0);
        }

        if ($oldSectionId <= 0) {
            $oldSectionId = (int)($placement['sectionId'] ?? 0);
        }

        $column = (int)($block['column'] ?? 0);

        if ($column <= 0) {
            $column = (int)($props['column'] ?? 0);
        }

        if ($column <= 0) {
            $column = (int)($placement['column'] ?? 0);
        }

        if ($column <= 0) {
            $column = 1;
        }

        $column = max(1, min(4, $column));

        $newSectionId = 0;

        if ($oldSectionId > 0 && isset($sectionIdMap[$oldSectionId])) {
            $newSectionId = (int)$sectionIdMap[$oldSectionId];
        }

        if ($newSectionId > 0) {
            $props['sectionId'] = $newSectionId;
            $props['column'] = $column;
            $props['_placement'] = [
                'sectionId' => $newSectionId,
                'column' => $column,
            ];
        } else {
            unset($props['sectionId'], $props['column'], $props['_placement']);
        }

        $newBlock = [
            'id' => $nextBlockId++,
            'pageId' => (int)$pageIdMap[$oldPageId],
            'type' => (string)($block['type'] ?? 'text'),
            'sort' => (int)($block['sort'] ?? 500),
            'content' => self::sanitizeDiskData($block['content'] ?? []),
            'props' => $props,
            'createdBy' => $userId,
            'createdAt' => $now,
            'updatedBy' => $userId,
            'updatedAt' => $now,
        ];

        if ($newSectionId > 0) {
            $newBlock['sectionId'] = $newSectionId;
            $newBlock['column'] = $column;
        }

        $blocks[] = sb_normalize_block_record($newBlock);
    }

    sb_write_blocks($blocks);
}


---

6. Что важно после правки

Старые уже созданные шаблоны не исправятся, потому что в них секции уже не попали.

После правки нужно:

1. Открыть исходный сайт, где секции настроены правильно.


2. Нажать “Сохранить как шаблон” заново.


3. Создать сайт из нового шаблона.



Тогда секции сохранят:

количество колонок
gap
container
padding
фон
привязку блоков к секциям и колонкам

Потом проверь созданный сайт: публичная страница должна сразу открыть секции с теми же колонками, что были у шаблона.