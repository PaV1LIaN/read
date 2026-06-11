Да, повторяю.

Идея правильная: сейчас у тебя компоненты частично лежат тут:

/local/sitebuilder/views/blocks/

А disk уже живёт нормально как отдельный компонент:

/local/sitebuilder/components/disk/

Нужно привести всё к единой структуре:

/local/sitebuilder/components/
    heading/
        render.php
    text/
        render.php
    button/
        render.php
    html/
        render.php
    spacer/
        render.php
    table/
        render.php
    disk/
        render.php
        class.php
        template.php
        ...

Старую папку:

/local/sitebuilder/views/blocks/

пока не удаляем. Сделаем fallback: если новый компонент не найден, сайт берёт старый файл.


---

1. Правим public_render.php

Файл:

/local/sitebuilder/lib/public_render.php

Найди функцию:

sb_public_render_block

И замени её блок на это:

if (!function_exists('sb_public_component_render_file')) {
    function sb_public_component_render_file(string $type): string
    {
        $type = strtolower(trim($type));
        $type = preg_replace('/[^a-z0-9_-]/i', '', $type);

        if ($type === '') {
            $type = 'text';
        }

        $root = dirname(__DIR__);

        $candidates = [
            $root . '/components/' . $type . '/render.php',
            $root . '/views/blocks/' . $type . '.php',
        ];

        foreach ($candidates as $file) {
            if (is_file($file)) {
                return $file;
            }
        }

        $fallbacks = [
            $root . '/components/text/render.php',
            $root . '/views/blocks/text.php',
        ];

        foreach ($fallbacks as $file) {
            if (is_file($file)) {
                return $file;
            }
        }

        throw new RuntimeException('SiteBuilder component renderer not found: ' . $type);
    }
}

if (!function_exists('sb_public_render_block')) {
    function sb_public_render_block(array $block, array $context = []): string
    {
        $block = sb_normalize_block_record($block);

        $type = (string)($block['type'] ?? 'text');
        $template = sb_public_component_render_file($type);

        $content = (array)($block['content'] ?? []);
        $props = (array)($block['props'] ?? []);

        ob_start();
        include $template;
        return (string)ob_get_clean();
    }
}

Теперь рендер будет искать так:

1. /components/table/render.php
2. /views/blocks/table.php
3. /components/text/render.php
4. /views/blocks/text.php


---

2. Создай папки

Создай:

/local/sitebuilder/components/heading/
/local/sitebuilder/components/text/
/local/sitebuilder/components/button/
/local/sitebuilder/components/html/
/local/sitebuilder/components/spacer/
/local/sitebuilder/components/table/

Папка disk уже есть.


---

3. Компонент heading

Файл:

/local/sitebuilder/components/heading/render.php

Код:

<?php

$level = strtolower((string)($content['level'] ?? 'h2'));

if (!in_array($level, ['h1', 'h2', 'h3', 'h4', 'h5', 'h6'], true)) {
    $level = 'h2';
}

$text = sb_public_h((string)($content['text'] ?? ''));

$align = (string)($content['align'] ?? 'left');

if (!in_array($align, ['left', 'center', 'right'], true)) {
    $align = 'left';
}
?>

<section class="sb-block sb-block--heading">
    <<?= $level ?> class="sb-heading sb-heading--<?= sb_public_h($level) ?>" style="text-align:<?= sb_public_h($align) ?>;">
        <?= $text ?>
    </<?= $level ?>>
</section>


---

4. Компонент text

Файл:

/local/sitebuilder/components/text/render.php

Код:

<?php

$html = (string)($content['html'] ?? '');
?>

<section class="sb-block sb-block--text">
    <div class="sb-block__inner sb-text">
        <?= $html ?>
    </div>
</section>


---

5. Компонент button

Файл:

/local/sitebuilder/components/button/render.php

Код:

<?php

$text = sb_public_h((string)($content['text'] ?? 'Кнопка'));
$href = sb_public_h((string)($content['href'] ?? '#'));
$target = sb_public_h((string)($content['target'] ?? '_self'));

$align = (string)($content['align'] ?? 'left');

if (!in_array($align, ['left', 'center', 'right'], true)) {
    $align = 'left';
}
?>

<section class="sb-block sb-block--button">
    <div class="sb-button-wrap" style="text-align:<?= sb_public_h($align) ?>;">
        <a class="sb-button" href="<?= $href ?>" target="<?= $target ?>">
            <?= $text ?>
        </a>
    </div>
</section>


---

6. Компонент html

Файл:

/local/sitebuilder/components/html/render.php

Код:

<?php

$html = (string)($content['html'] ?? '');
?>

<section class="sb-block sb-block--html">
    <div class="sb-block__inner">
        <?= $html ?>
    </div>
</section>


---

7. Компонент spacer

Файл:

/local/sitebuilder/components/spacer/render.php

Код:

<?php

$height = max(0, (int)($content['height'] ?? 30));
?>

<div class="sb-block sb-block--spacer" style="height:<?= $height ?>px;"></div>


---

8. Компонент disk

Файл:

/local/sitebuilder/components/disk/render.php

Код:

<?php

global $USER;

$site = $context['site'] ?? [];
$currentPage = $context['currentPage'] ?? [];

$siteId = (int)($site['id'] ?? 0);
$pageId = (int)($currentPage['id'] ?? 0);
$blockId = (int)($block['id'] ?? 0);

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/components/disk/class.php';

$component = new SitebuilderDiskComponent([
    'SITE_ID' => $siteId,
    'PAGE_ID' => $pageId,
    'BLOCK_ID' => $blockId,
    'CURRENT_USER_ID' => is_object($USER) ? (int)$USER->GetID() : 0,
]);

$component->execute();


---

9. Компонент table

Таблица у тебя сейчас большая и рабочая, поэтому руками её лучше не переписывать.

Просто скопируй весь файл:

/local/sitebuilder/views/blocks/table.php

в новый файл:

/local/sitebuilder/components/table/render.php

Старый файл пока оставь.


---

10. Проверка

После этого открой public-страницу.

Должны работать:

heading
text
button
html
spacer
table
disk

Если какой-то новый компонент не найден, рендер возьмёт старый файл из:

/local/sitebuilder/views/blocks/


---

Что это даст

Было:

views/blocks/heading.php
views/blocks/text.php
views/blocks/button.php
views/blocks/table.php

Станет:

components/heading/render.php
components/text/render.php
components/button/render.php
components/table/render.php
components/disk/render.php

Следующим шагом можно будет разнести стили и JS по компонентам:

components/table/styles.css
components/table/edit.js
components/table/view.js

components/disk/styles.css
components/disk/script.js

Но сначала лучше безопасно перенести только render.php, чтобы не сломать текущую таблицу и диск.