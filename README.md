Отлично. Делаем template-пакет.

Сейчас закроем:

template.list

template.createFromPage

template.applyToPage

template.rename

template.delete


Логика будет простая и практичная:

шаблон хранится в templates.json

шаблон принадлежит сайту

шаблон содержит набор блоков страницы

при создании из страницы копируются блоки этой страницы

при применении к странице её текущие блоки заменяются блоками шаблона


Это хороший рабочий вариант без усложнений.


---

1. Обнови /local/sitebuilder/lib/helpers.php

Добавь в конец файла:

<?php

if (!function_exists('sb_find_template')) {
    function sb_find_template(int $templateId): ?array
    {
        foreach (sb_read_templates() as $tpl) {
            if ((int)($tpl['id'] ?? 0) === $templateId) {
                return $tpl;
            }
        }
        return null;
    }
}

if (!function_exists('sb_next_template_id')) {
    function sb_next_template_id(array $templates = null): int
    {
        if ($templates === null) {
            $templates = sb_read_templates();
        }

        $maxId = 0;
        foreach ($templates as $tpl) {
            $maxId = max($maxId, (int)($tpl['id'] ?? 0));
        }

        return $maxId + 1;
    }
}

if (!function_exists('sb_templates_for_site')) {
    function sb_templates_for_site(int $siteId): array
    {
        $templates = array_values(array_filter(sb_read_templates(), static function ($tpl) use ($siteId) {
            return (int)($tpl['siteId'] ?? 0) === $siteId;
        }));

        usort($templates, static function ($a, $b) {
            return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
        });

        return $templates;
    }
}

if (!function_exists('sb_normalize_template_record')) {
    function sb_normalize_template_record(array $tpl): array
    {
        if (!isset($tpl['name'])) {
            $tpl['name'] = '';
        }
        if (!isset($tpl['siteId'])) {
            $tpl['siteId'] = 0;
        }
        if (!isset($tpl['blocks']) || !is_array($tpl['blocks'])) {
            $tpl['blocks'] = [];
        }

        $tpl['blocks'] = array_map('sb_normalize_block_record', $tpl['blocks']);

        usort($tpl['blocks'], static function ($a, $b) {
            $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);
            if ($sortCmp !== 0) {
                return $sortCmp;
            }
            return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
        });

        return $tpl;
    }
}


---

2. Полностью замени /local/sitebuilder/api/handlers/template.php

<?php

global $USER;

if ($action === 'template.list') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    sb_require_viewer($siteId);

    $templates = array_map('sb_normalize_template_record', sb_templates_for_site($siteId));

    sb_json_ok([
        'templates' => $templates,
    ]);
}

if ($action === 'template.createFromPage') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $pageId = (int)($_POST['pageId'] ?? 0);
    $name = trim((string)($_POST['name'] ?? ''));

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }
    if ($pageId <= 0) {
        sb_json_error('PAGE_ID_REQUIRED', 422);
    }
    if ($name === '') {
        sb_json_error('NAME_REQUIRED', 422);
    }

    sb_require_editor($siteId);

    $page = sb_find_page($pageId);
    if (!$page || (int)($page['siteId'] ?? 0) !== $siteId) {
        sb_json_error('PAGE_NOT_IN_SITE', 422);
    }

    $pageBlocks = sb_blocks_for_page($pageId);

    $storedBlocks = [];
    foreach ($pageBlocks as $block) {
        $copy = sb_normalize_block_record($block);
        unset($copy['pageId']);
        $storedBlocks[] = $copy;
    }

    $templates = sb_read_templates();

    $template = [
        'id' => sb_next_template_id($templates),
        'siteId' => $siteId,
        'name' => $name,
        'sourcePageId' => $pageId,
        'blocks' => $storedBlocks,
        'createdBy' => (int)$USER->GetID(),
        'createdAt' => date('c'),
        'updatedAt' => date('c'),
        'updatedBy' => (int)$USER->GetID(),
    ];

    $templates[] = $template;
    sb_write_templates($templates);

    sb_json_ok([
        'template' => sb_normalize_template_record($template),
    ]);
}

if ($action === 'template.applyToPage') {
    $templateId = (int)($_POST['templateId'] ?? 0);
    $pageId = (int)($_POST['pageId'] ?? 0);

    if ($templateId <= 0) {
        sb_json_error('TEMPLATE_ID_REQUIRED', 422);
    }
    if ($pageId <= 0) {
        sb_json_error('PAGE_ID_REQUIRED', 422);
    }

    $template = sb_find_template($templateId);
    if (!$template) {
        sb_json_error('TEMPLATE_NOT_FOUND', 404);
    }

    $page = sb_find_page($pageId);
    if (!$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    if ((int)($template['siteId'] ?? 0) !== $siteId) {
        sb_json_error('TEMPLATE_NOT_IN_SITE', 422);
    }

    sb_require_editor($siteId);

    $blocks = sb_read_blocks();

    $blocks = array_values(array_filter($blocks, static function ($b) use ($pageId) {
        return (int)($b['pageId'] ?? 0) !== $pageId;
    }));

    $nextBlockId = sb_next_block_id($blocks);

    $templateBlocks = $template['blocks'] ?? [];
    usort($templateBlocks, static function ($a, $b) {
        $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);
        if ($sortCmp !== 0) {
            return $sortCmp;
        }
        return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
    });

    $sort = 10;
    foreach ($templateBlocks as $tplBlock) {
        $newBlock = sb_normalize_block_record($tplBlock);
        $newBlock['id'] = $nextBlockId++;
        $newBlock['pageId'] = $pageId;
        $newBlock['sort'] = $sort;
        $newBlock['createdBy'] = (int)$USER->GetID();
        $newBlock['createdAt'] = date('c');
        $newBlock['updatedAt'] = date('c');
        $newBlock['updatedBy'] = (int)$USER->GetID();
        $blocks[] = $newBlock;
        $sort += 10;
    }

    sb_write_blocks($blocks);

    sb_json_ok([
        'blocks' => array_map('sb_normalize_block_record', sb_blocks_for_page($pageId)),
    ]);
}

if ($action === 'template.rename') {
    $id = (int)($_POST['id'] ?? 0);
    $name = trim((string)($_POST['name'] ?? ''));

    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }
    if ($name === '') {
        sb_json_error('NAME_REQUIRED', 422);
    }

    $template = sb_find_template($id);
    if (!$template) {
        sb_json_error('TEMPLATE_NOT_FOUND', 404);
    }

    $siteId = (int)($template['siteId'] ?? 0);
    sb_require_editor($siteId);

    $templates = sb_read_templates();
    $updated = null;

    foreach ($templates as &$tpl) {
        if ((int)($tpl['id'] ?? 0) === $id) {
            $tpl['name'] = $name;
            $tpl['updatedAt'] = date('c');
            $tpl['updatedBy'] = (int)$USER->GetID();
            $updated = $tpl;
            break;
        }
    }
    unset($tpl);

    if (!$updated) {
        sb_json_error('TEMPLATE_NOT_FOUND', 404);
    }

    sb_write_templates($templates);

    sb_json_ok([
        'template' => sb_normalize_template_record($updated),
    ]);
}

if ($action === 'template.delete') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }

    $template = sb_find_template($id);
    if (!$template) {
        sb_json_error('TEMPLATE_NOT_FOUND', 404);
    }

    $siteId = (int)($template['siteId'] ?? 0);
    sb_require_editor($siteId);

    $templates = sb_read_templates();
    $before = count($templates);

    $templates = array_values(array_filter($templates, static function ($tpl) use ($id) {
        return (int)($tpl['id'] ?? 0) !== $id;
    }));

    if (count($templates) === $before) {
        sb_json_error('TEMPLATE_NOT_FOUND', 404);
    }

    sb_write_templates($templates);

    sb_json_ok();
}

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'template',
    'action' => $action,
]);


---

3. Что теперь должно работать

После вставки:

template.list

template.createFromPage

template.applyToPage

template.rename

template.delete



---

4. Как сейчас выглядит шаблон

Пример записи в templates.json:

{
  "id": 1,
  "siteId": 3,
  "name": "Лендинг с hero",
  "sourcePageId": 12,
  "blocks": [
    {
      "id": 101,
      "type": "heading",
      "sort": 10,
      "content": {
        "text": "Заголовок",
        "level": "h1",
        "align": "center"
      },
      "props": {}
    },
    {
      "id": 102,
      "type": "text",
      "sort": 20,
      "content": {
        "html": "<p>Описание</p>"
      },
      "props": {}
    }
  ]
}

Обрати внимание:

id у блоков внутри шаблона хранятся как снимок

при применении к странице им назначаются новые реальные block.id


Это правильное поведение.


---

5. Что проверить руками

Проверь по порядку:

1. создать пару блоков на странице


2. вызвать template.createFromPage


3. вызвать template.list


4. переименовать шаблон через template.rename


5. создать новую страницу


6. применить шаблон через template.applyToPage


7. проверить, что блоки появились на новой странице


8. удалить шаблон




---

6. Важный момент по apply

Сейчас template.applyToPage делает простое и безопасное поведение:

удаляет все текущие блоки страницы

вставляет блоки из шаблона


Это лучше, чем пытаться “умно мерджить”, потому что мердж почти всегда ломает редактор.


---

7. Что дальше лучше делать

Теперь у тебя закрыты:

site

page

block

menu

access

template


Остаются два крупных участка:

layout

file / Disk


Я бы сейчас делал layout-пакет, потому что он ближе к редактору и публичному рендеру, а Disk лучше выделять уже отдельным шагом.

Напиши: давай layout-пакет