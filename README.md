В public_page.php ты вставил почти правильно.

Этот файл у тебя сейчас:

правильно определяет $pageHasDiskBlock

правильно подключает styles.css

правильно подключает core.js и disk/script.js


То есть проблема не здесь.

Ошибка EMPTY_SESSID почти наверняка осталась потому, что не обновлен файл блока или не обновлен script.js.


---

Что проверить обязательно

1. Проверь /local/sitebuilder/views/blocks/disk.php

Именно в нем должен быть data-sessid.

Файл должен быть вот таким:

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
         data-sessid="<?= htmlspecialchars(bitrix_sessid(), ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>"
         data-initial-state="<?= htmlspecialchars(json_encode([
             'siteId' => $siteId,
             'pageId' => $pageId,
             'blockId' => $blockId,
             'settings' => $diskProps,
         ], JSON_UNESCAPED_UNICODE), ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>">
    </div>
</div>

Самое важное:

Должна быть строка:

data-sessid="<?= htmlspecialchars(bitrix_sessid(), ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>"

Если ее нет, script.js не сможет взять sessid.


---

2. Проверь /local/sitebuilder/components/disk/script.js

Функция getSessid() должна быть именно такой:

DiskComponent.prototype.getSessid = function () {
  if (window.BX && typeof BX.bitrix_sessid === 'function') {
    return BX.bitrix_sessid();
  }

  var sessidFromData = this.root.getAttribute('data-sessid');
  if (sessidFromData) {
    return String(sessidFromData);
  }

  return '';
};

Если у тебя там по-прежнему:

return '';

то будет EMPTY_SESSID.


---

3. Быстрая проверка прямо в браузере

Открой страницу с диском, нажми F12, Console и выполни:

document.querySelector('.sb-disk')?.getAttribute('data-sessid')

Что должно быть

Должна вернуться строка sessid, например:

"c6f4b8d1a2..."

Если вернется

null

""


значит проблема точно в views/blocks/disk.php.


---

4. Вторая проверка

В консоли выполни:

typeof BX !== 'undefined' && typeof BX.bitrix_sessid === 'function' ? BX.bitrix_sessid() : 'NO_BX'

Что должно быть

Либо строка sessid, либо NO_BX.

Если NO_BX, это не критично, если есть data-sessid.


---

По твоему public_page.php

Вот этот кусок у тебя нормальный:

<?php if ($pageHasDiskBlock): ?>
<script src="/bitrix/js/main/core/core.js"></script>
<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js"></script>
<?php endif; ?>

Оставляй его.


---

Где сейчас почти наверняка проблема

С вероятностью почти 100% одно из двух:

1. в views/blocks/disk.php нет data-sessid


2. в script.js не обновлена функция getSessid()




---

Что сделать сейчас

Проверь эти два файла и потом сразу выполни в консоли:

document.querySelector('.sb-disk')?.outerHTML

Если хочешь, пришли сюда:

содержимое views/blocks/disk.php

текущую функцию getSessid() из script.js


Я сразу скажу, где именно не совпало.