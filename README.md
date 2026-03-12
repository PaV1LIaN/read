Идём к left/right.

Сейчас сделаем следующий этап в public.php:

если showLeft включён, выводить блоки зоны left

если showRight включён, выводить блоки зоны right

основную страницу ставить между ними

ширины брать из layout.leftWidth и layout.rightWidth


Что меняем

1. Загрузи блоки left/right

В public.php найди этот кусок:

$headerBlocks = !empty($layout['showHeader']) ? sb_layout_zone_blocks($siteId, 'header') : [];
$footerBlocks = !empty($layout['showFooter']) ? sb_layout_zone_blocks($siteId, 'footer') : [];

И замени на:

$headerBlocks = !empty($layout['showHeader']) ? sb_layout_zone_blocks($siteId, 'header') : [];
$footerBlocks = !empty($layout['showFooter']) ? sb_layout_zone_blocks($siteId, 'footer') : [];
$leftBlocks   = !empty($layout['showLeft'])   ? sb_layout_zone_blocks($siteId, 'left')   : [];
$rightBlocks  = !empty($layout['showRight'])  ? sb_layout_zone_blocks($siteId, 'right')  : [];

$leftWidth = (int)($layout['leftWidth'] ?? 260);
$rightWidth = (int)($layout['rightWidth'] ?? 260);

if ($leftWidth < 160) $leftWidth = 160;
if ($leftWidth > 500) $leftWidth = 500;
if ($rightWidth < 160) $rightWidth = 160;
if ($rightWidth > 500) $rightWidth = 500;


---

2. Подготовь HTML для боковых зон

Найди этот кусок:

$pageHtml = sb_render_blocks($blocks, $siteId);
$headerHtml = sb_render_blocks($headerBlocks, $siteId);
$footerHtml = sb_render_blocks($footerBlocks, $siteId);

И замени на:

$pageHtml = sb_render_blocks($blocks, $siteId);
$headerHtml = sb_render_blocks($headerBlocks, $siteId);
$footerHtml = sb_render_blocks($footerBlocks, $siteId);
$leftHtml = sb_render_blocks($leftBlocks, $siteId);
$rightHtml = sb_render_blocks($rightBlocks, $siteId);


---

3. Добавь стили для трёхколоночного layout

В <style> перед .meta или после .layoutInner добавь:

.layoutMainGrid{
  display:grid;
  gap:20px;
  align-items:start;
}

.layoutSidebar{
  min-width:0;
}

.layoutContent{
  min-width:0;
}

@media (max-width: 980px){
  .layoutMainGrid{
    grid-template-columns:1fr !important;
  }
}


---

4. Замени центральный вывод страницы

Сейчас у тебя внутри <div class="wrap"> есть такой кусок:

<?php if ($headerHtml !== ''): ?>
  <div class="layoutHeader">
    <div class="layoutInner">
      <?= $headerHtml ?>
    </div>
  </div>
<?php endif; ?>

<?= $pageHtml ?>

<?php if ($footerHtml !== ''): ?>
  <div class="layoutFooter">
    <div class="layoutInner">
      <?= $footerHtml ?>
    </div>
  </div>
<?php endif; ?>

Его нужно заменить на это:

<?php if ($headerHtml !== ''): ?>
  <div class="layoutHeader">
    <div class="layoutInner">
      <?= $headerHtml ?>
    </div>
  </div>
<?php endif; ?>

<?php
  $gridCols = '1fr';
  if ($leftHtml !== '' && $rightHtml !== '') {
      $gridCols = $leftWidth . 'px 1fr ' . $rightWidth . 'px';
  } elseif ($leftHtml !== '') {
      $gridCols = $leftWidth . 'px 1fr';
  } elseif ($rightHtml !== '') {
      $gridCols = '1fr ' . $rightWidth . 'px';
  }
?>
<div class="layoutMainGrid" style="grid-template-columns: <?=h($gridCols)?>;">
  <?php if ($leftHtml !== ''): ?>
    <aside class="layoutSidebar layoutSidebarLeft">
      <?= $leftHtml ?>
    </aside>
  <?php endif; ?>

  <main class="layoutContent">
    <?= $pageHtml ?>
  </main>

  <?php if ($rightHtml !== ''): ?>
    <aside class="layoutSidebar layoutSidebarRight">
      <?= $rightHtml ?>
    </aside>
  <?php endif; ?>
</div>

<?php if ($footerHtml !== ''): ?>
  <div class="layoutFooter">
    <div class="layoutInner">
      <?= $footerHtml ?>
    </div>
  </div>
<?php endif; ?>


---

Что получится

Теперь public.php будет собираться так:

hero

header zone

grid:

left

page content

right


footer zone



---

Как проверить

1. Открой layout.php?siteId=...


2. Включи showLeft


3. В зоне Left создай Text, например LEFT TEST


4. Включи showRight


5. В зоне Right создай Text, например RIGHT TEST


6. Открой публичную страницу



Ожидаемо:

слева появится LEFT TEST

справа RIGHT TEST

в центре останется страница

на узком экране всё уйдёт в одну колонку


Следующим шагом я бы сделал так, чтобы header/footer/left/right можно было открывать из index.php одной кнопкой Layout.