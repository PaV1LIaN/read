Отлично. Делаем left как вертикальное меню сайта.

Самый правильный путь без слома текущего:

добавить настройку leftMode

если leftMode = menu, то слева рендерить menuItems вертикально

если leftMode = blocks, оставить текущее поведение


Так ты получишь:

left как настоящее левое меню

при желании можно вернуть обычные блоки


Шаг 1. Добавь leftMode в layout сайта

Где

В api.php, в месте где создаётся сайт, у тебя уже есть:

'layout' => [
    'showHeader' => true,
    'showFooter' => true,
    'showLeft' => false,
    'showRight' => false,
    'leftWidth' => 260,
    'rightWidth' => 260,
],

Замени на

'layout' => [
    'showHeader' => true,
    'showFooter' => true,
    'showLeft' => false,
    'showRight' => false,
    'leftWidth' => 260,
    'rightWidth' => 260,
    'leftMode' => 'blocks',
],


---

Шаг 2. Разреши сохранять leftMode

Где

В api.php, внутри action:

if ($action === 'layout.updateSettings') {

Найди место, где обновляются leftWidth/rightWidth, и перед блоком с $layout += [...] добавь:

if (array_key_exists('leftMode', $_POST)) {
    $leftMode = trim((string)$_POST['leftMode']);
    if (!in_array($leftMode, ['blocks', 'menu'], true)) {
        $leftMode = 'blocks';
    }
    $layout['leftMode'] = $leftMode;
}

И в массиве дефолтов:

$layout += [
    'showHeader' => true,
    'showFooter' => true,
    'showLeft' => false,
    'showRight' => false,
    'leftWidth' => 260,
    'rightWidth' => 260,
];

замени на

$layout += [
    'showHeader' => true,
    'showFooter' => true,
    'showLeft' => false,
    'showRight' => false,
    'leftWidth' => 260,
    'rightWidth' => 260,
    'leftMode' => 'blocks',
];


---

Шаг 3. Добавь выбор режима в layout.php

Где

В layout.php, в левой карточке с настройками layout, после поля Ширина left или рядом с left-настройками добавь:

<div class="field">
  <label>Режим left</label>
  <select id="leftMode" class="input">
    <option value="blocks">Обычные блоки</option>
    <option value="menu">Меню сайта</option>
  </select>
</div>


---

Где ещё

В fillSettingsForm() добавь:

document.getElementById('leftMode').value = layoutSettings.leftMode || 'blocks';


---

И ещё

В saveLayoutSettings() добавь в payload:

leftMode: document.getElementById('leftMode').value || 'blocks',

То есть кусок станет таким:

const res = await api('layout.updateSettings', {
  siteId,
  showHeader: document.getElementById('showHeader').checked ? '1' : '0',
  showFooter: document.getElementById('showFooter').checked ? '1' : '0',
  showLeft: document.getElementById('showLeft').checked ? '1' : '0',
  showRight: document.getElementById('showRight').checked ? '1' : '0',
  leftWidth: parseInt(document.getElementById('leftWidth').value || '260', 10),
  rightWidth: parseInt(document.getElementById('rightWidth').value || '260', 10),
  leftMode: document.getElementById('leftMode').value || 'blocks',
});


---

Шаг 4. Поддержка leftMode в public.php

Где

В public.php, в месте где у тебя сейчас:

$layout = (isset($site['layout']) && is_array($site['layout'])) ? $site['layout'] : [
    'showHeader' => true,
    'showFooter' => true,
    'showLeft' => false,
    'showRight' => false,
    'leftWidth' => 260,
    'rightWidth' => 260,
];

замени на

$layout = (isset($site['layout']) && is_array($site['layout'])) ? $site['layout'] : [
    'showHeader' => true,
    'showFooter' => true,
    'showLeft' => false,
    'showRight' => false,
    'leftWidth' => 260,
    'rightWidth' => 260,
    'leftMode' => 'blocks',
];

$leftMode = (string)($layout['leftMode'] ?? 'blocks');
if (!in_array($leftMode, ['blocks', 'menu'], true)) $leftMode = 'blocks';


---

Шаг 5. Если leftMode = menu, не использовать left-блоки

Где

Найди в public.php:

$headerBlocks = !empty($layout['showHeader']) ? sb_layout_zone_blocks($siteId, 'header') : [];
$footerBlocks = !empty($layout['showFooter']) ? sb_layout_zone_blocks($siteId, 'footer') : [];
$leftBlocks   = !empty($layout['showLeft'])   ? sb_layout_zone_blocks($siteId, 'left')   : [];
$rightBlocks  = !empty($layout['showRight'])  ? sb_layout_zone_blocks($siteId, 'right')  : [];

замени на

$headerBlocks = !empty($layout['showHeader']) ? sb_layout_zone_blocks($siteId, 'header') : [];
$footerBlocks = !empty($layout['showFooter']) ? sb_layout_zone_blocks($siteId, 'footer') : [];
$leftBlocks   = (!empty($layout['showLeft']) && $leftMode === 'blocks') ? sb_layout_zone_blocks($siteId, 'left') : [];
$rightBlocks  = !empty($layout['showRight']) ? sb_layout_zone_blocks($siteId, 'right') : [];


---

Шаг 6. Собери HTML для left menu

Где

После строки:

$menuItems = [];

ничего не надо.
А вот после того как меню уже собрано, то есть после блока:

if ($mainMenu && is_array($mainMenu['items'] ?? null)) {
   ...
}

добавь:

$leftMenuHtml = '';
if (!empty($layout['showLeft']) && $leftMode === 'menu' && count($menuItems)) {
    ob_start();
    ?>
    <nav class="sideMenu">
      <?php foreach ($menuItems as $mi): ?>
        <a class="sideMenuLink <?= !empty($mi['active']) ? 'active' : '' ?>"
           href="<?=h($mi['href'])?>"
           <?=preg_match('~^https?://~i', $mi['href']) ? 'target="_blank" rel="noopener noreferrer"' : ''?>>
          <?=h($mi['title'])?>
        </a>
      <?php endforeach; ?>
    </nav>
    <?php
    $leftMenuHtml = (string)ob_get_clean();
}


---

Шаг 7. Выбери, что рендерить слева

Где

Найди:

$leftHtml = sb_render_blocks($leftBlocks, $siteId);
$rightHtml = sb_render_blocks($rightBlocks, $siteId);

замени на

$leftHtml = ($leftMode === 'menu') ? $leftMenuHtml : sb_render_blocks($leftBlocks, $siteId);
$rightHtml = sb_render_blocks($rightBlocks, $siteId);


---

Шаг 8. Стили для левого меню

Где

В <style> public.php добавь:

.layoutSidebarBox{
  background:#fff;
  border:1px solid var(--line);
  border-radius:16px;
  padding:14px;
  box-shadow: var(--shadow);
  position: sticky;
  top: 88px;
}

.layoutSidebarBox .block:first-child{
  margin-top:0;
}

.sideMenu{
  display:flex;
  flex-direction:column;
  gap:8px;
}

.sideMenuLink{
  display:block;
  padding:10px 12px;
  border-radius:12px;
  border:1px solid var(--line);
  background:#fff;
  color:var(--text);
  text-decoration:none;
  line-height:1.35;
}

.sideMenuLink:hover{
  text-decoration:none;
  border-color:#d1d5db;
  background:#f9fafb;
}

.sideMenuLink.active{
  background: color-mix(in srgb, var(--sb-accent) 12%, #fff);
  border-color: color-mix(in srgb, var(--sb-accent) 28%, #e5e7eb);
  color: var(--sb-accent);
  font-weight:700;
}


---

Шаг 9. Оберни left/right в sidebar box

Где

В public.php, там где выводятся left/right, замени на такой вариант:

<?php if ($leftHtml !== ''): ?>
  <aside class="layoutSidebar layoutSidebarLeft">
    <div class="layoutSidebarBox">
      <?= $leftHtml ?>
    </div>
  </aside>
<?php endif; ?>

<main class="layoutContent">
  <?= $pageHtml ?>
</main>

<?php if ($rightHtml !== ''): ?>
  <aside class="layoutSidebar layoutSidebarRight">
    <div class="layoutSidebarBox">
      <?= $rightHtml ?>
    </div>
  </aside>
<?php endif; ?>


---

Что получится

Теперь:

если leftMode = blocks, слева будут обычные layout-блоки

если leftMode = menu, слева будет вертикальное меню сайта

активная страница будет подсвечена


Как проверить

1. Открой layout.php?siteId=...


2. Включи showLeft


3. Выбери Режим left = Меню сайта


4. Сохрани


5. Открой public.php



Слева должно появиться аккуратное вертикальное меню, а не просто текстовый блок.

Если хочешь, следующим сообщением я соберу тебе полностью готовый layout.php и public.php уже с leftMode=menu, чтобы ты просто заменил файлы.