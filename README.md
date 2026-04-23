Причина найдена: блок уже рендерится, но bootstrap режется на CSRF-проверке, потому что на публичной странице не передается sessid.

Ошибка:

{
  "error": "SERVER_ERROR",
  "message": "EMPTY_SESSID"
}

означает, что в script.js this.getSessid() вернул пустую строку.


---

Почему так

На публичной странице, скорее всего, не подключен Bitrix core, поэтому это не работает:

BX.bitrix_sessid()

А disk.bootstrap требует sessid, потому что в bootstrap.php у тебя вызывается:

DiskCsrf::validateFromRequest();


---

Как починить

Есть 2 обязательных правки.


---

1. Передать sessid в HTML блока

Файл:

/local/sitebuilder/views/blocks/disk.php

Замени его содержимое на это:

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


---

2. Исправить getSessid() в script.js

Файл:

/local/sitebuilder/components/disk/script.js

Найди функцию:

DiskComponent.prototype.getSessid = function () {
  if (window.BX && typeof BX.bitrix_sessid === 'function') {
    return BX.bitrix_sessid();
  }
  return '';
};

И замени на:

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


---

Желательно еще сделать

3. Подключить Bitrix core на публичной странице

Файл:

/local/sitebuilder/views/layout/public_page.php

Перед подключением disk/script.js добавь:

<?php if ($pageHasDiskBlock): ?>
<script src="/bitrix/js/main/core/core.js"></script>
<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js"></script>
<?php endif; ?>

Если у тебя сейчас подключается только disk/script.js, то замени этот кусок на код выше.


---

После этого

Обнови страницу и снова посмотри api.php?action=bootstrap.

Ожидаемо должно быть уже не EMPTY_SESSID, а либо:

успешный ответ ok: true

либо следующая уже предметная ошибка по root/правам/контексту



---

Что проверить сразу

В DevTools → Network → bootstrap → Payload должен появиться:

"sessid": "..."

Если хочешь, следующим сообщением я скажу, какой будет следующий самый вероятный затык после sessid, чтобы сразу подготовить фикс.