Идём дальше: подключаем header/footer в public.php.

Сейчас задача такая:

загрузить layout сайта,

взять zones.header и zones.footer,

отрисовать их до и после основной страницы.


Что меняем

Нужно править public.php в 3 местах:

1. добавить чтение layouts.json,


2. добавить рендер layout-блоков,


3. обернуть основной контент в header/main/footer.




---

1. Добавь helpers в public.php

Рядом с твоими функциями чтения sites/pages/blocks добавь:

function sb_public_read_layouts(): array {
    $path = $_SERVER['DOCUMENT_ROOT'] . '/upload/sitebuilder/layouts.json';
    if (!file_exists($path)) return [];
    $raw = file_get_contents($path);
    $data = json_decode((string)$raw, true);
    return is_array($data) ? $data : [];
}

function sb_public_find_layout_record(int $siteId): ?array {
    foreach (sb_public_read_layouts() as $r) {
        if ((int)($r['siteId'] ?? 0) === $siteId) return $r;
    }
    return null;
}

function sb_public_layout_zone_blocks(int $siteId, string $zone): array {
    $rec = sb_public_find_layout_record($siteId);
    if (!$rec) return [];

    $zones = is_array($rec['zones'] ?? null) ? $rec['zones'] : [];
    $blocks = is_array($zones[$zone] ?? null) ? $zones[$zone] : [];
    usort($blocks, fn($a,$b) => (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500));
    return $blocks;
}


---

2. Если у тебя уже есть функция рендера обычных блоков страницы

Например что-то вроде:

render_block($block, $site, ...)

то лучше переиспользовать её.

То есть layout-блоки должны рендериться той же функцией, что и обычные page blocks.

Если у тебя сейчас рендер идёт прямо циклом, лучше вынести его в одну функцию. Пример:

function sb_public_render_blocks(array $blocks, array $site): string {
    ob_start();
    foreach ($blocks as $block) {
        echo sb_public_render_block($block, $site);
    }
    return (string)ob_get_clean();
}

Если у тебя уже есть sb_public_render_block(...), используй её.


---

3. Загрузи layout в public.php

После того как ты уже определил $site и $page, добавь:

$layout = is_array($site['layout'] ?? null) ? $site['layout'] : [
    'showHeader' => true,
    'showFooter' => true,
    'showLeft' => false,
    'showRight' => false,
    'leftWidth' => 260,
    'rightWidth' => 260,
];

$headerBlocks = !empty($layout['showHeader']) ? sb_public_layout_zone_blocks((int)$site['id'], 'header') : [];
$footerBlocks = !empty($layout['showFooter']) ? sb_public_layout_zone_blocks((int)$site['id'], 'footer') : [];


---

4. Подготовь HTML для header/footer

Если у тебя есть рендер блоков функцией, добавь:

$headerHtml = sb_public_render_blocks($headerBlocks, $site);
$footerHtml = sb_public_render_blocks($footerBlocks, $site);

Если рендер у тебя не вынесен, временно можно сделать так:

ob_start();
foreach ($headerBlocks as $block) {
    echo sb_public_render_block($block, $site);
}
$headerHtml = (string)ob_get_clean();

ob_start();
foreach ($footerBlocks as $block) {
    echo sb_public_render_block($block, $site);
}
$footerHtml = (string)ob_get_clean();


---

5. В HTML-шаблоне public.php вставь header/footer

Там, где сейчас выводится только основной контент страницы, сделай структуру:

<?php if ($headerHtml !== ''): ?>
  <header class="sb-layout-header">
    <div class="sb-layout-inner">
      <?= $headerHtml ?>
    </div>
  </header>
<?php endif; ?>

<main class="sb-layout-main">
  <div class="sb-layout-inner">
    <?= $pageHtml ?>
  </div>
</main>

<?php if ($footerHtml !== ''): ?>
  <footer class="sb-layout-footer">
    <div class="sb-layout-inner">
      <?= $footerHtml ?>
    </div>
  </footer>
<?php endif; ?>

Где $pageHtml — это уже отрендеренные блоки самой страницы.


---

6. Добавь базовые стили в public.php

В <style> добавь:

.sb-layout-header,
.sb-layout-footer {
  width: 100%;
}

.sb-layout-main {
  width: 100%;
}

.sb-layout-inner {
  width: 100%;
  max-width: var(--sb-container-width, 1100px);
  margin: 0 auto;
  box-sizing: border-box;
}

.sb-layout-header {
  margin-bottom: 24px;
}

.sb-layout-footer {
  margin-top: 24px;
}

И если у тебя контейнерная ширина сайта уже есть, пробрось её в CSS variable:

<?php
$containerWidth = (int)($site['settings']['containerWidth'] ?? 1100);
?>
<style>
  :root {
    --sb-container-width: <?= $containerWidth ?>px;
  }
</style>


---

Что проверить

После этого:

1. зайди в layout.php?siteId=...


2. в зоне Header создай Text блок, например Это header


3. в зоне Footer создай Text блок, например Это footer


4. открой публичную страницу сайта



Ожидаемое поведение:

сверху появляется header,

снизу появляется footer,

основная страница между ними.



---

Самое важное замечание

public.php должен рендерить layout-блоки без привязки к pageId, только по siteId.

То есть:

страница даёт main content,

layout даёт global content.



---

Следующим шагом после этого будет уже подключение left/right через grid-контейнер.