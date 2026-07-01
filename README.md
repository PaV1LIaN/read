Понял. Тут надо проверить не диск, а цепочку рендера блока `text`.

Скорее всего одно из трёх:

```text id="90webz"
1. Блок text сохранился, но public_render его не рендерит.
2. Нет файла /components/text/render.php.
3. Текст сохранился в props.content/text, а render.php читает другое поле.
```

Сначала сделай самый надёжный фикс.

---

# 1. Создай / замени `components/text/render.php`

Файл:

```text id="o5d2h6"
/local/sitebuilder/components/text/render.php
```

Код:

```php id="pr8xjf"
<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

/*
 * Универсальный render для текстового блока.
 * Поддерживает разные варианты хранения:
 * props.text, props.content, props.html, block.text, block.content.
 */

$block = is_array($block ?? null) ? $block : [];
$props = is_array($block['props'] ?? null) ? $block['props'] : [];

$text = '';

if (array_key_exists('text', $props)) {
    $text = (string)$props['text'];
} elseif (array_key_exists('content', $props)) {
    $text = (string)$props['content'];
} elseif (array_key_exists('html', $props)) {
    $text = (string)$props['html'];
} elseif (array_key_exists('text', $block)) {
    $text = (string)$block['text'];
} elseif (array_key_exists('content', $block)) {
    $text = (string)$block['content'];
}

$text = trim($text);

$align = (string)($props['align'] ?? 'left');
$size = (string)($props['size'] ?? '16');
$color = (string)($props['color'] ?? '#111827');
$lineHeight = (string)($props['lineHeight'] ?? '1.6');
$maxWidth = (string)($props['maxWidth'] ?? '');

if (!in_array($align, ['left', 'center', 'right', 'justify'], true)) {
    $align = 'left';
}

$sizeNumber = (int)$size;

if ($sizeNumber <= 0) {
    $sizeNumber = 16;
}

$allowedTags = '<br><b><strong><i><em><u><s><p><span><ul><ol><li><a>';

$safeText = strip_tags($text, $allowedTags);

$style = [
    'text-align:' . $align,
    'font-size:' . $sizeNumber . 'px',
    'line-height:' . htmlspecialchars($lineHeight, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8'),
    'color:' . htmlspecialchars($color, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8'),
];

if ($maxWidth !== '') {
    $style[] = 'max-width:' . htmlspecialchars($maxWidth, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
}

?>

<div class="sb-text-block" style="<?= implode(';', $style) ?>">
    <?= $safeText !== '' ? nl2br($safeText) : '' ?>
</div>
```

---

# 2. Создай CSS для текста

Файл:

```text id="ty7v42"
/local/sitebuilder/components/text/styles.css
```

Код:

```css id="4vz4dy"
.sb-text-block {
    display: block;
    width: 100%;
    margin: 0;
    padding: 0;
    word-break: break-word;
}

.sb-text-block p {
    margin: 0 0 12px;
}

.sb-text-block p:last-child {
    margin-bottom: 0;
}

.sb-text-block a {
    color: inherit;
    text-decoration: underline;
}
```

---

# 3. Проверь, что public_render регистрирует тип

Файл:

```text id="1v7xcu"
/local/sitebuilder/lib/public_render.php
```

В функции:

```php id="hbljm7"
sb_public_render_block()
```

должно быть что-то такое:

```php id="2uc7iq"
$type = sb_public_component_type((string)($block['type'] ?? ''));

sb_public_register_component_type($type);

$renderFile = sb_public_component_render_file($type);
```

Если `sb_public_register_component_type($type);` нет — добавь перед include render-файла.

---

# 4. Проверь, что layout подключает assets компонентов

Файл:

```text id="13tz17"
/local/sitebuilder/views/layout/public_page.php
```

После вывода блоков должно быть:

```php id="e1xpcm"
<?= sb_public_render_component_assets($basePath, $isPublicEditMode) ?>
```

Если этого нет — добавь после цикла рендера блоков.

---

# 5. Быстрая проверка через исходный код страницы

Открой опубликованную страницу сайта и нажми:

```text id="ya6v3u"
Ctrl + U
```

Найди:

```text id="v9r5kk"
sb-text-block
```

Если `sb-text-block` есть, но текста не видно — проблема в CSS/цвете/высоте блока.

Если `sb-text-block` нет — проблема в том, что блок `text` вообще не доходит до public render.

---

# 6. Если после этого всё равно не появится

Тогда нужно проверить, как именно блок сохранился. Вставь в консоль редактора:

```js id="j79045"
(async function () {
  const pageId = window.SB_PAGE_ID || window.pageId || null;

  console.log('pageId:', pageId);

  if (!pageId) {
    console.warn('Не нашёл pageId глобально. Открой Network и посмотри ответ page.get/block.list.');
  }

  console.log('Ищи блок type=text в данных страницы/блоков.');
})();
```

Но сначала сделай `render.php` выше. Очень похоже, что у компонента `text` просто нет нормального публичного рендера или он читает не то поле.
