Значит одна из правок не встала в нужное место. Давай быстро локализуем без гадания.

Проверь по пунктам.

1. В public.php временно выведи текущий режим

Сразу после:

$leftMode = (string)($layout['leftMode'] ?? 'blocks');
if (!in_array($leftMode, ['blocks', 'menu'], true)) $leftMode = 'blocks';

вставь:

echo '<pre style="padding:10px;background:#fff3cd;border:1px solid #eed;">leftMode=' . h($leftMode) . ' | showLeft=' . (!empty($layout['showLeft']) ? '1' : '0') . '</pre>';

Если на странице увидишь:

leftMode=blocks — значит настройка не сохраняется из layout.php

leftMode=menu | showLeft=0 — значит левый сайдбар выключен

leftMode=menu | showLeft=1 — значит проблема уже в рендере меню



---

2. Проверь, сохраняется ли leftMode из layout.php

В layout.php в saveLayoutSettings() должен быть:

leftMode: document.getElementById('leftMode').value || 'blocks',

И в fillSettingsForm():

document.getElementById('leftMode').value = layoutSettings.leftMode || 'blocks';

И в HTML должен быть сам select:

<select id="leftMode" class="input">
  <option value="blocks">Обычные блоки</option>
  <option value="menu">Меню сайта</option>
</select>

Если этого нет хотя бы в одном месте, режим не будет работать.


---

3. Проверь api.php

В layout.updateSettings должен быть код:

if (array_key_exists('leftMode', $_POST)) {
    $leftMode = trim((string)$_POST['leftMode']);
    if (!in_array($leftMode, ['blocks', 'menu'], true)) {
        $leftMode = 'blocks';
    }
    $layout['leftMode'] = $leftMode;
}

И в дефолтах:

'leftMode' => 'blocks',

Если этого нет, layout.php будет отправлять значение, но api.php его просто не сохранит.


---

4. Самая частая ошибка в public.php

Вот этот кусок должен идти после того, как уже собран $menuItems:

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

Если ты вставил его до формирования $menuItems, меню будет пустым.


---

5. Итоговый выбор левого HTML

В public.php должно быть именно так:

$leftHtml = ($leftMode === 'menu') ? $leftMenuHtml : sb_render_blocks($leftBlocks, $siteId);
$rightHtml = sb_render_blocks($rightBlocks, $siteId);

Если там осталось старое:

$leftHtml = sb_render_blocks($leftBlocks, $siteId);

тогда всегда будут обычные блоки.


---

6. В выводе страницы слева должен быть sidebar box

Нужно вот так:

<?php if ($leftHtml !== ''): ?>
  <aside class="layoutSidebar layoutSidebarLeft">
    <div class="layoutSidebarBox">
      <?= $leftHtml ?>
    </div>
  </aside>
<?php endif; ?>


---

Сейчас быстрее всего будет так: пришли мне 4 куска кода:

1. layout.php — fillSettingsForm()


2. layout.php — saveLayoutSettings()


3. api.php — кусок layout.updateSettings


4. public.php — место, где формируется $menuItems, $leftMenuHtml, $leftHtml



По ним я сразу скажу, где именно сломано.