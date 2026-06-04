Нашли точное место. Проблема вот здесь:

$content = (array)($block['content'] ?? []);
$props = (array)($block['props'] ?? []);

Если content приходит как JSON-строка:

{"text":"Новый текстовый блок"}

то (array) не превращает её в массив с ключом text, а делает примерно так:

[
    0 => '{"text":"Новый текстовый блок"}'
]

Поэтому views/blocks/text.php не видит $content['text'].

Что сделать

В файле:

/local/sitebuilder/lib/public_render.php

после функции sb_public_h() добавь:

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

Должно получиться так:

if (!function_exists('sb_public_h')) {
    function sb_public_h(string $s): string
    {
        return htmlspecialchars($s, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
    }
}

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

Потом найди функцию:

if (!function_exists('sb_public_render_block')) {
    function sb_public_render_block(array $block, array $context = []): string

И замени её полностью на эту:

if (!function_exists('sb_public_render_block')) {
    function sb_public_render_block(array $block, array $context = []): string
    {
        $type = (string)($block['type'] ?? 'text');
        $template = dirname(__DIR__) . '/views/blocks/' . $type . '.php';

        if (!file_exists($template)) {
            $template = dirname(__DIR__) . '/views/blocks/text.php';
        }

        $block = sb_normalize_block_record($block);

        $content = sb_public_to_array($block['content'] ?? []);
        $props = sb_public_to_array($block['props'] ?? []);

        ob_start();
        include $template;
        return (string)ob_get_clean();
    }
}

Если после этого текст всё равно не появится

Тогда проблема уже в самом шаблоне текстового блока.

Проверь файл:

/local/sitebuilder/views/blocks/text.php

Он должен быть примерно такой:

<?php
$text = (string)($content['text'] ?? '');

if ($text === '') {
    return;
}
?>

<div class="sb-public-block sb-public-block--text">
    <div class="sb-public-text">
        <?= nl2br(sb_public_h($text)) ?>
    </div>
</div>

И в CSS можно добавить:

.sb-public-block--text {
    margin: 12px 0;
}

.sb-public-text {
    color: #374151;
    font-size: 16px;
    line-height: 1.65;
}

После замены обнови публичную страницу через Ctrl + F5. Текстовый компонент должен начать отображаться.