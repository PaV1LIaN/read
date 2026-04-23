Посмотрел архив. У тебя не switch по типам блоков, а другая схема:

блок рендерится через lib/public_render.php

там функция sb_public_render_block() просто подключает шаблон из
views/blocks/<type>.php

сама публичная страница собирается в
views/layout/public_page.php


То есть пункт 6 нужно вставлять не в один файл, а так:


---

Куда именно вставлять

1. Рендер самого блока disk

Нужно создать новый файл:

/local/sitebuilder/views/blocks/disk.php

Потому что в архиве блоки рендерятся именно так:

views/blocks/button.php

views/blocks/heading.php

views/blocks/html.php

views/blocks/text.php


И lib/public_render.php сам подхватит disk.php, если type = 'disk'.


---

2. Подключение CSS/JS диска на публичной странице

Это нужно вставлять в:

/local/sitebuilder/views/layout/public_page.php

Именно там находится <head> и финальная сборка HTML страницы.


---

Что именно сделать


---

Шаг 1. Создай файл /local/sitebuilder/views/blocks/disk.php

Вставь туда вот это:

<?php
$site = $context['site'] ?? [];
$currentPage = $context['currentPage'] ?? [];
$siteId = (int)($site['id'] ?? 0);
$pageId = (int)($currentPage['id'] ?? 0);
$blockId = (int)($block['id'] ?? 0);
$diskProps = is_array($props ?? null) ? $props : [];
?>
<div class="sb-block sb-block--disk">
    <div class="sb-disk"
         data-site-id="<?= $siteId ?>"
         data-page-id="<?= $pageId ?>"
         data-block-id="<?= $blockId ?>"
         data-initial-state="<?= htmlspecialchars(json_encode([
             'siteId' => $siteId,
             'pageId' => $pageId,
             'blockId' => $blockId,
             'settings' => $diskProps,
         ], JSON_UNESCAPED_UNICODE), ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>">
    </div>
</div>


---

Шаг 2. В views/layout/public_page.php добавить проверку, есть ли disk на странице

Найди вот этот участок:

$pageHtml = sb_public_render_blocks($pageBlocks, $vm);
$menuHtml = sb_public_render_menu($menu, $basePath, $siteId);

И сразу после него вставь:

$pageHasDiskBlock = false;

foreach ($pageBlocks as $pageBlock) {
    if ((string)($pageBlock['type'] ?? '') === 'disk') {
        $pageHasDiskBlock = true;
        break;
    }
}

if (!$pageHasDiskBlock) {
    foreach ($headerBlocks as $layoutBlock) {
        if ((string)($layoutBlock['type'] ?? '') === 'disk') {
            $pageHasDiskBlock = true;
            break;
        }
    }
}

if (!$pageHasDiskBlock) {
    foreach ($footerBlocks as $layoutBlock) {
        if ((string)($layoutBlock['type'] ?? '') === 'disk') {
            $pageHasDiskBlock = true;
            break;
        }
    }
}

if (!$pageHasDiskBlock) {
    foreach ($leftBlocks as $layoutBlock) {
        if ((string)($layoutBlock['type'] ?? '') === 'disk') {
            $pageHasDiskBlock = true;
            break;
        }
    }
}

if (!$pageHasDiskBlock) {
    foreach ($rightBlocks as $layoutBlock) {
        if ((string)($layoutBlock['type'] ?? '') === 'disk') {
            $pageHasDiskBlock = true;
            break;
        }
    }
}


---

Шаг 3. В том же views/layout/public_page.php подключить стили диска

Найди в <head> строку:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css">

И сразу после нее вставь:

<?php if ($pageHasDiskBlock): ?>
    <link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/components/disk/styles.css">
<?php endif; ?>


---

Шаг 4. В views/layout/public_page.php подключить JS диска

Найди внизу перед </body> существующий <script> и после него вставь:

<?php if ($pageHasDiskBlock): ?>
<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js"></script>
<?php endif; ?>


---

Что важно

Тебе не нужно вставлять case 'disk' в public.php, потому что:

public.php только собирает view model и подключает layout

сам рендер блоков идет через:

lib/public_render.php

views/blocks/*.php



То есть в твоем архиве правильная точка интеграции — новый файл views/blocks/disk.php, а не switch.


---

Итого по файлам

Создать:

/local/sitebuilder/views/blocks/disk.php

Изменить:

/local/sitebuilder/views/layout/public_page.php


---

Если совсем коротко

6 пункт в твоем архиве нужно реализовывать через новый файл views/blocks/disk.php, а не вставкой в public.php.

Если хочешь, следующим сообщением я пришлю тебе готовый патч именно для public_page.php целиком с уже вставленными кусками, чтобы ты просто заменил нужные строки без ручного поиска.