Скорее всего проблема такая же, как была с колонками: у блока text поле content приходит как JSON-строка, а публичный рендер ждёт массив. Поэтому он не находит content['text'].

Нужно поправить рендер блока text.

1. Найди функцию рендера блоков

Ищи в проекте:

function sb_public_render_blocks

или:

function sb_public_render_block

Скорее всего она находится в одном из файлов:

/local/sitebuilder/public.php
/local/sitebuilder/views/layout/public_page.php
/local/sitebuilder/lib/helpers.php


---

2. Добавь универсальную функцию декодирования

Если ты уже добавлял sb_public_to_array() в public_page.php, повторно не надо.

Она должна быть такой:

if (!function_exists('sb_public_to_array')) {
    function sb_public_to_array($value): array
    {
        if (is_array($value)) {
            return $value;
        }

        if (is_string($value) && trim($value) !== '') {
            $decoded = json_decode($value, true);

            if (is_array($decoded)) {
                return $decoded;
            }
        }

        return [];
    }
}


---

3. Исправь рендер блока text

Найди в функции рендера блоков место примерно такое:

if ($type === 'text') {
    ...
}

И замени обработку text на такую:

if ($type === 'text') {
    $content = sb_public_to_array($block['content'] ?? []);
    $props = sb_public_to_array($block['props'] ?? []);

    $text = (string)($content['text'] ?? '');

    if ($text === '') {
        return '';
    }

    return '<div class="sb-public-block sb-public-block--text">'
        . '<div class="sb-public-text">'
        . nl2br(sb_public_h($text))
        . '</div>'
        . '</div>';
}


---

4. Заодно лучше поправить heading, button, html

Чтобы потом не всплыла такая же проблема, в каждом типе блока content и props лучше получать так:

$content = sb_public_to_array($block['content'] ?? []);
$props = sb_public_to_array($block['props'] ?? []);

Например для heading:

if ($type === 'heading') {
    $content = sb_public_to_array($block['content'] ?? []);

    $text = (string)($content['text'] ?? '');

    if ($text === '') {
        return '';
    }

    return '<div class="sb-public-block sb-public-block--heading">'
        . '<h2 class="sb-public-heading">'
        . sb_public_h($text)
        . '</h2>'
        . '</div>';
}


---

5. Добавь CSS, если текст есть, но его почти не видно

Файл:

/local/sitebuilder/assets/public/public.css

В конец:

.sb-public-block--text {
    margin: 12px 0;
}

.sb-public-text {
    color: #374151;
    font-size: 16px;
    line-height: 1.65;
    white-space: normal;
}

.sb-public-heading {
    margin: 0 0 14px;
    color: #111827;
    font-size: 28px;
    line-height: 1.25;
    font-weight: 900;
}

И обнови версию CSS в public_page.php:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=4">


---

После этого текстовый блок должен появиться на публичной странице. Если не появится — пришли мне функцию sb_public_render_blocks / sb_public_render_block, я точечно заменю её целиком.