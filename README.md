<?php
define('NO_KEEP_STATISTIC', true);
define('NO_AGENT_STATISTIC', true);
define('DisableEventsCheck', true);

require $_SERVER['DOCUMENT_ROOT'].'/bitrix/modules/main/include/prolog_before.php';

global $USER, $APPLICATION;

if (!$USER->IsAuthorized()) {
    LocalRedirect('/auth/');
}

header('Content-Type: text/html; charset=UTF-8');

\Bitrix\Main\UI\Extension::load([
    'main.core',
    'ui.buttons',
    'ui.dialogs.messagebox',
    'ui.notification',
]);

$siteId = (int)($_GET['siteId'] ?? 0);
$pageId = (int)($_GET['pageId'] ?? 0);
?>
<!doctype html>
<html lang="ru">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Редактор блоков</title>
  <?php $APPLICATION->ShowHead(); ?>
  <style>
    body { font-family: Arial, sans-serif; margin:0; background:#f6f7f8; }
    .top { background:#fff; border-bottom:1px solid #e5e7ea; padding:12px 16px; display:flex; gap:10px; align-items:center; flex-wrap:wrap; }
    .content { padding: 18px; }
    .card { background:#fff; border:1px solid #e5e7ea; border-radius:12px; padding:16px; }
    .muted { color:#6a737f; }
    .block { border:1px solid #eee; border-radius:10px; padding:12px; margin-top:10px; }
    .row { display:flex; gap:8px; align-items:center; justify-content:space-between; flex-wrap:wrap; }
    .btns { display:flex; gap:6px; flex-wrap:wrap; }
    pre { white-space:pre-wrap; margin:10px 0 0; background:#f9fafb; border:1px solid #eee; border-radius:8px; padding:10px; }
    a { color:#0b57d0; text-decoration:none; }
    a:hover { text-decoration:underline; }
    code { background:#f3f4f6; padding:2px 6px; border-radius:6px; }
    .imgPrev { margin-top:10px; max-width: 420px; border:1px solid #eee; border-radius:10px; overflow:hidden; background:#fafafa; }
    .imgPrev img { display:block; width:100%; height:auto; }
    .field { margin-top:10px; }
    .field label { display:block; font-size:12px; color:#6a737f; margin-bottom:4px; }
    .input, select, textarea { width:100%; padding:8px; border:1px solid #d0d7de; border-radius:8px; box-sizing:border-box; }

    .btnPreview { margin-top:10px; display:inline-block; padding:10px 14px; border-radius:10px; border:1px solid #e5e7ea; text-decoration:none; }
    .btnPrimary { background:#2563eb; color:#fff; border-color:#2563eb; }
    .btnSecondary { background:#fff; color:#111; }

    .headingPreview { margin-top:10px; border:1px dashed #e5e7ea; border-radius:10px; padding:10px; }
    .headingPreview h1, .headingPreview h2, .headingPreview h3 { margin:0; }

    .colsPreview { margin-top:10px; border:1px dashed #e5e7ea; border-radius:10px; padding:10px; display:grid; gap:10px; }
    .colsPreview .cell { background:#fafafa; border:1px solid #eee; border-radius:10px; padding:10px; min-height:48px; }
    .colsPreview pre { margin:0; background:transparent; border:none; padding:0; }
  </style>
</head>
<body>
  <div class="top">
    <a href="/local/sitebuilder/index.php">← Назад</a>
    <div class="muted">Редактор блоков</div>
    <div class="muted">|</div>
    <div><b>siteId:</b> <code><?= (int)$siteId ?></code></div>
    <div><b>pageId:</b> <code><?= (int)$pageId ?></code></div>

    <div style="flex:1;"></div>

    <a class="ui-btn ui-btn-light" target="_blank"
       href="/local/sitebuilder/view.php?siteId=<?= (int)$siteId ?>&pageId=<?= (int)$pageId ?>">Открыть просмотр</a>

    <a class="ui-btn ui-btn-light" target="_blank"
       href="/local/sitebuilder/files.php?siteId=<?= (int)$siteId ?>">Файлы</a>
  </div>

  <div class="content">
    <div class="card">
      <div class="row">
        <div class="muted">Блоки: <b>Text</b>, <b>Image</b>, <b>Button</b>, <b>Heading</b>, <b>Columns2</b>.</div>
        <div class="btns">
          <button class="ui-btn ui-btn-primary" id="btnAddText">+ Text</button>
          <button class="ui-btn ui-btn-primary" id="btnAddImage">+ Image</button>
          <button class="ui-btn ui-btn-primary" id="btnAddButton">+ Button</button>
          <button class="ui-btn ui-btn-primary" id="btnAddHeading">+ Heading</button>
          <button class="ui-btn ui-btn-primary" id="btnAddCols2">+ Columns2</button>
        </div>
      </div>

      <div id="blocksBox" style="margin-top:12px;"></div>
    </div>
  </div>

<script>
BX.ready(function () {
  const siteId = <?= (int)$siteId ?>;
  const pageId = <?= (int)$pageId ?>;

  const blocksBox = document.getElementById('blocksBox');
  const btnAddText = document.getElementById('btnAddText');
  const btnAddImage = document.getElementById('btnAddImage');
  const btnAddButton = document.getElementById('btnAddButton');
  const btnAddHeading = document.getElementById('btnAddHeading');
  const btnAddCols2 = document.getElementById('btnAddCols2');

  function api(action, data) {
    return new Promise((resolve, reject) => {
      BX.ajax({
        url: '/local/sitebuilder/api.php',
        method: 'POST',
        dataType: 'json',
        data: Object.assign({ action, sessid: BX.bitrix_sessid() }, data || {}),
        onsuccess: resolve,
        onfailure: reject
      });
    });
  }

  function fileDownloadUrl(fileId) {
    return `/local/sitebuilder/download.php?siteId=${siteId}&fileId=${fileId}`;
  }

  async function getFilesForSite() {
    const res = await api('file.list', { siteId });
    if (!res || res.ok !== true) throw new Error(res?.error || 'file.list failed');
    return res.files || [];
  }

  function btnClass(variant) {
    return (variant === 'secondary') ? 'btnPreview btnSecondary' : 'btnPreview btnPrimary';
  }
  function headingTag(level) {
    if (level === 'h1' || level === 'h2' || level === 'h3') return level;
    return 'h2';
  }
  function headingAlign(align) {
    if (align === 'left' || align === 'center' || align === 'right') return align;
    return 'left';
  }

  function colsGridTemplate(ratio) {
    if (ratio === '33-67') return '1fr 2fr';
    if (ratio === '67-33') return '2fr 1fr';
    return '1fr 1fr';
  }

  function renderBlocks(blocks) {
    if (!blocks || !blocks.length) {
      blocksBox.innerHTML = '<div class="muted">Блоков нет. Добавь первый.</div>';
      return;
    }

    blocksBox.innerHTML = blocks.map(b => {
      const type = b.type || '';
      const sort = b.sort;
      const id = b.id;

      const commonBtns = `
        <button class="ui-btn ui-btn-light ui-btn-xs" data-move-block-id="${id}" data-move-dir="up">↑</button>
        <button class="ui-btn ui-btn-light ui-btn-xs" data-move-block-id="${id}" data-move-dir="down">↓</button>
      `;

      if (type === 'text') {
        const text = (b.content && typeof b.content.text === 'string') ? b.content.text : '';
        return `
          <div class="block">
            <div class="row">
              <div><b>#${id}</b> <span class="muted">(text | sort ${sort})</span></div>
              <div class="btns">
                ${commonBtns}
                <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-text-id="${id}">Редактировать</button>
                <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
              </div>
            </div>
            <pre>${BX.util.htmlspecialchars(text)}</pre>
          </div>
        `;
      }

      if (type === 'image') {
        const fileId = b.content && b.content.fileId ? parseInt(b.content.fileId, 10) : 0;
        const alt = b.content && typeof b.content.alt === 'string' ? b.content.alt : '';
        const img = fileId
          ? `<div class="imgPrev"><img src="${fileDownloadUrl(fileId)}" alt="${BX.util.htmlspecialchars(alt)}"></div>`
          : '<div class="muted" style="margin-top:10px;">Файл не выбран</div>';

        return `
          <div class="block">
            <div class="row">
              <div><b>#${id}</b> <span class="muted">(image | sort ${sort} | fileId ${fileId || '-'})</span></div>
              <div class="btns">
                ${commonBtns}
                <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-image-id="${id}">Редактировать</button>
                <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
              </div>
            </div>
            <div class="muted" style="margin-top:8px;">alt: ${BX.util.htmlspecialchars(alt)}</div>
            ${img}
          </div>
        `;
      }

      if (type === 'button') {
        const text = (b.content && typeof b.content.text === 'string') ? b.content.text : '';
        const url = (b.content && typeof b.content.url === 'string') ? b.content.url : '';
        const variant = (b.content && typeof b.content.variant === 'string') ? b.content.variant : 'primary';

        return `
          <div class="block">
            <div class="row">
              <div><b>#${id}</b> <span class="muted">(button | sort ${sort} | ${BX.util.htmlspecialchars(variant)})</span></div>
              <div class="btns">
                ${commonBtns}
                <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-button-id="${id}">Редактировать</button>
                <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
              </div>
            </div>
            <div class="muted" style="margin-top:8px;">url: ${BX.util.htmlspecialchars(url)}</div>
            <a class="${btnClass(variant)}" href="${BX.util.htmlspecialchars(url)}" target="_blank" rel="noopener noreferrer">
              ${BX.util.htmlspecialchars(text)}
            </a>
          </div>
        `;
      }

      if (type === 'heading') {
        const text = (b.content && typeof b.content.text === 'string') ? b.content.text : '';
        const level = (b.content && typeof b.content.level === 'string') ? b.content.level : 'h2';
        const align = (b.content && typeof b.content.align === 'string') ? b.content.align : 'left';
        const tag = headingTag(level);
        const al = headingAlign(align);

        return `
          <div class="block">
            <div class="row">
              <div><b>#${id}</b> <span class="muted">(heading | sort ${sort} | ${BX.util.htmlspecialchars(tag)} | ${BX.util.htmlspecialchars(al)})</span></div>
              <div class="btns">
                ${commonBtns}
                <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-heading-id="${id}">Редактировать</button>
                <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
              </div>
            </div>

            <div class="headingPreview" style="text-align:${BX.util.htmlspecialchars(al)};">
              <${tag}>${BX.util.htmlspecialchars(text)}</${tag}>
            </div>
          </div>
        `;
      }

      if (type === 'columns2') {
        const left = (b.content && typeof b.content.left === 'string') ? b.content.left : '';
        const right = (b.content && typeof b.content.right === 'string') ? b.content.right : '';
        const ratio = (b.content && typeof b.content.ratio === 'string') ? b.content.ratio : '50-50';
        const tpl = colsGridTemplate(ratio);

        return `
          <div class="block">
            <div class="row">
              <div><b>#${id}</b> <span class="muted">(columns2 | sort ${sort} | ratio ${BX.util.htmlspecialchars(ratio)})</span></div>
              <div class="btns">
                ${commonBtns}
                <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-cols2-id="${id}">Редактировать</button>
                <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
              </div>
            </div>

            <div class="colsPreview" style="grid-template-columns:${tpl};">
              <div class="cell"><pre>${BX.util.htmlspecialchars(left)}</pre></div>
              <div class="cell"><pre>${BX.util.htmlspecialchars(right)}</pre></div>
            </div>
          </div>
        `;
      }

      return `
        <div class="block">
          <div class="row">
            <div><b>#${id}</b> <span class="muted">(unknown)</span></div>
            <div class="btns">
              <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
            </div>
          </div>
          <div class="muted" style="margin-top:10px;">Неизвестный тип: ${BX.util.htmlspecialchars(type)}</div>
        </div>
      `;
    }).join('');
  }

  function loadBlocks() {
    api('block.list', { pageId }).then(res => {
      if (!res || res.ok !== true) {
        BX.UI.Notification.Center.notify({ content: 'Не удалось загрузить блоки' });
        return;
      }
      renderBlocks(res.blocks);
    }).catch(() => BX.UI.Notification.Center.notify({ content: 'Ошибка block.list' }));
  }

  // ===== create blocks =====

  function addTextBlock() {
    BX.UI.Dialogs.MessageBox.show({
      title: 'Новый Text блок',
      message: '<textarea id="new_text" style="width:100%;height:140px;padding:8px;border:1px solid #d0d7de;border-radius:8px;"></textarea>',
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mb) {
        const text = document.getElementById('new_text')?.value ?? '';
        api('block.create', { pageId, type: 'text', text })
          .then(res => {
            if (!res || res.ok !== true) { BX.UI.Notification.Center.notify({ content: 'Не удалось создать блок' }); return; }
            BX.UI.Notification.Center.notify({ content: 'Блок создан' });
            mb.close(); loadBlocks();
          })
          .catch(() => BX.UI.Notification.Center.notify({ content: 'Ошибка block.create' }));
      }
    });
  }

  function addImageBlock() {
    BX.UI.Dialogs.MessageBox.show({
      title: 'Новый Image блок',
      message: `
        <div>
          <div class="muted">Выбери файл из “Файлы” этого сайта.</div>
          <div class="field">
            <label>Файл</label>
            <select id="img_file" class="input">
              <option value="">Загрузка списка...</option>
            </select>
          </div>
          <div class="field">
            <label>ALT</label>
            <input id="img_alt" class="input" placeholder="например: Логотип" />
          </div>
          <div id="img_preview" class="imgPrev" style="display:none;">
            <img id="img_preview_img" src="" alt="">
          </div>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mb) {
        const fileId = parseInt(document.getElementById('img_file')?.value || '0', 10);
        const alt = (document.getElementById('img_alt')?.value || '').trim();
        if (!fileId) { BX.UI.Notification.Center.notify({ content: 'Выбери файл' }); return; }

        api('block.create', { pageId, type: 'image', fileId, alt })
          .then(res => {
            if (!res || res.ok !== true) { BX.UI.Notification.Center.notify({ content: 'Не удалось создать image-блок' }); return; }
            BX.UI.Notification.Center.notify({ content: 'Image-блок создан' });
            mb.close(); loadBlocks();
          })
          .catch(() => BX.UI.Notification.Center.notify({ content: 'Ошибка block.create (image)' }));
      }
    });

    setTimeout(async function () {
      const sel = document.getElementById('img_file');
      if (!sel) return;

      try {
        const files = await getFilesForSite();
        if (!files.length) { sel.innerHTML = '<option value="">Файлов нет (загрузите в “Файлы”)</option>'; return; }

        sel.innerHTML = '<option value="">— Выберите файл —</option>' + files.map(f =>
          `<option value="${f.id}">${BX.util.htmlspecialchars(f.name)} (${f.id})</option>`
        ).join('');

        sel.addEventListener('change', function () {
          const id = parseInt(sel.value || '0', 10);
          const wrap = document.getElementById('img_preview');
          const img = document.getElementById('img_preview_img');
          if (!wrap || !img) return;
          if (!id) { wrap.style.display = 'none'; img.src = ''; return; }
          wrap.style.display = 'block';
          img.src = fileDownloadUrl(id);
        });
      } catch (e) {
        sel.innerHTML = '<option value="">Ошибка загрузки файлов</option>';
        BX.UI.Notification.Center.notify({ content: 'Не удалось получить список файлов' });
      }
    }, 0);
  }

  function addButtonBlock() {
    BX.UI.Dialogs.MessageBox.show({
      title: 'Новый Button блок',
      message: `
        <div>
          <div class="field">
            <label>Текст кнопки</label>
            <input id="btn_text" class="input" placeholder="например: Купить" />
          </div>
          <div class="field">
            <label>URL</label>
            <input id="btn_url" class="input" placeholder="https://... или /local/..." />
          </div>
          <div class="field">
            <label>Вариант</label>
            <select id="btn_variant" class="input">
              <option value="primary">primary</option>
              <option value="secondary">secondary</option>
            </select>
          </div>
          <div class="muted" style="margin-top:10px;">Превью:</div>
          <a id="btn_preview" class="btnPreview btnPrimary" href="#" target="_blank" rel="noopener noreferrer">Кнопка</a>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mb) {
        const text = (document.getElementById('btn_text')?.value || '').trim();
        const url  = (document.getElementById('btn_url')?.value || '').trim();
        const variant = (document.getElementById('btn_variant')?.value || 'primary');

        if (!text) { BX.UI.Notification.Center.notify({ content: 'Введите текст' }); return; }
        if (!url)  { BX.UI.Notification.Center.notify({ content: 'Введите URL' }); return; }

        api('block.create', { pageId, type: 'button', text, url, variant })
          .then(res => {
            if (!res || res.ok !== true) { BX.UI.Notification.Center.notify({ content: 'Не удалось создать button-блок' }); return; }
            BX.UI.Notification.Center.notify({ content: 'Button-блок создан' });
            mb.close(); loadBlocks();
          })
          .catch(() => BX.UI.Notification.Center.notify({ content: 'Ошибка block.create (button)' }));
      }
    });
  }

  function addHeadingBlock() {
    BX.UI.Dialogs.MessageBox.show({
      title: 'Новый Heading блок',
      message: `
        <div>
          <div class="field">
            <label>Текст</label>
            <input id="h_text" class="input" placeholder="например: О нас" />
          </div>
          <div class="field">
            <label>Уровень</label>
            <select id="h_level" class="input">
              <option value="h1">h1</option>
              <option value="h2" selected>h2</option>
              <option value="h3">h3</option>
            </select>
          </div>
          <div class="field">
            <label>Выравнивание</label>
            <select id="h_align" class="input">
              <option value="left" selected>left</option>
              <option value="center">center</option>
              <option value="right">right</option>
            </select>
          </div>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mb) {
        const text = (document.getElementById('h_text')?.value || '').trim();
        const level = (document.getElementById('h_level')?.value || 'h2');
        const align = (document.getElementById('h_align')?.value || 'left');

        if (!text) { BX.UI.Notification.Center.notify({ content:'Введите текст' }); return; }

        api('block.create', { pageId, type:'heading', text, level, align })
          .then(res => {
            if (!res || res.ok !== true) { BX.UI.Notification.Center.notify({ content:'Не удалось создать heading' }); return; }
            BX.UI.Notification.Center.notify({ content:'Heading создан' });
            mb.close(); loadBlocks();
          })
          .catch(() => BX.UI.Notification.Center.notify({ content:'Ошибка block.create (heading)' }));
      }
    });
  }

  function addCols2Block() {
    BX.UI.Dialogs.MessageBox.show({
      title: 'Новый Columns2 блок',
      message: `
        <div>
          <div class="field">
            <label>Соотношение</label>
            <select id="c_ratio" class="input">
              <option value="50-50" selected>50 / 50</option>
              <option value="33-67">33 / 67</option>
              <option value="67-33">67 / 33</option>
            </select>
          </div>
          <div class="field">
            <label>Левая колонка (текст)</label>
            <textarea id="c_left" class="input" style="height:120px;"></textarea>
          </div>
          <div class="field">
            <label>Правая колонка (текст)</label>
            <textarea id="c_right" class="input" style="height:120px;"></textarea>
          </div>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mb) {
        const ratio = (document.getElementById('c_ratio')?.value || '50-50');
        const left = (document.getElementById('c_left')?.value || '');
        const right = (document.getElementById('c_right')?.value || '');

        api('block.create', { pageId, type:'columns2', ratio, left, right })
          .then(res => {
            if (!res || res.ok !== true) { BX.UI.Notification.Center.notify({ content:'Не удалось создать columns2' }); return; }
            BX.UI.Notification.Center.notify({ content:'Columns2 создан' });
            mb.close(); loadBlocks();
          })
          .catch(() => BX.UI.Notification.Center.notify({ content:'Ошибка block.create (columns2)' }));
      }
    });
  }

  // ===== edit blocks =====

  function editCols2Block(id) {
    api('block.list', { pageId }).then(res => {
      if (!res || res.ok !== true) return;
      const blk = (res.blocks || []).find(x => parseInt(x.id,10) === id);

      const curRatio = blk && blk.content ? (blk.content.ratio || '50-50') : '50-50';
      const curLeft = blk && blk.content ? (blk.content.left || '') : '';
      const curRight = blk && blk.content ? (blk.content.right || '') : '';

      BX.UI.Dialogs.MessageBox.show({
        title: 'Редактировать Columns2 #' + id,
        message: `
          <div>
            <div class="field">
              <label>Соотношение</label>
              <select id="ec_ratio" class="input">
                <option value="50-50" ${curRatio==='50-50'?'selected':''}>50 / 50</option>
                <option value="33-67" ${curRatio==='33-67'?'selected':''}>33 / 67</option>
                <option value="67-33" ${curRatio==='67-33'?'selected':''}>67 / 33</option>
              </select>
            </div>
            <div class="field">
              <label>Левая колонка (текст)</label>
              <textarea id="ec_left" class="input" style="height:120px;">${BX.util.htmlspecialchars(curLeft)}</textarea>
            </div>
            <div class="field">
              <label>Правая колонка (текст)</label>
              <textarea id="ec_right" class="input" style="height:120px;">${BX.util.htmlspecialchars(curRight)}</textarea>
            </div>
          </div>
        `,
        buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
        onOk: function (mb) {
          const ratio = (document.getElementById('ec_ratio')?.value || '50-50');
          const left = (document.getElementById('ec_left')?.value || '');
          const right = (document.getElementById('ec_right')?.value || '');

          api('block.update', { id, ratio, left, right })
            .then(r => {
              if (!r || r.ok !== true) { BX.UI.Notification.Center.notify({ content:'Не удалось сохранить columns2' }); return; }
              BX.UI.Notification.Center.notify({ content:'Сохранено' });
              mb.close(); loadBlocks();
            })
            .catch(() => BX.UI.Notification.Center.notify({ content:'Ошибка block.update (columns2)' }));
        }
      });
    });
  }

  // ===== common click handlers =====
  document.addEventListener('click', function (e) {
    const mvBtn = e.target.closest('[data-move-block-id]');
    if (mvBtn) {
      const id = parseInt(mvBtn.getAttribute('data-move-block-id'), 10);
      const dir = mvBtn.getAttribute('data-move-dir');
      api('block.move', { id, dir }).then(r => {
        if (!r || r.ok !== true) { BX.UI.Notification.Center.notify({ content: 'Не удалось переместить блок' }); return; }
        loadBlocks();
      }).catch(() => BX.UI.Notification.Center.notify({ content: 'Ошибка block.move' }));
      return;
    }

    const delBtn = e.target.closest('[data-del-block-id]');
    if (delBtn) {
      const id = parseInt(delBtn.getAttribute('data-del-block-id'), 10);
      BX.UI.Dialogs.MessageBox.show({
        title: 'Удалить блок #' + id + '?',
        message: 'Продолжить?',
        buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
        onOk: function (mb) {
          api('block.delete', { id }).then(r => {
            if (!r || r.ok !== true) { BX.UI.Notification.Center.notify({ content: 'Не удалось удалить блок' }); return; }
            BX.UI.Notification.Center.notify({ content: 'Удалено' });
            mb.close(); loadBlocks();
          }).catch(() => BX.UI.Notification.Center.notify({ content: 'Ошибка block.delete' }));
        }
      });
      return;
    }

    const ec = e.target.closest('[data-edit-cols2-id]');
    if (ec) { editCols2Block(parseInt(ec.getAttribute('data-edit-cols2-id'), 10)); return; }
  });

  btnAddText.addEventListener('click', addTextBlock);
  btnAddImage.addEventListener('click', addImageBlock);
  btnAddButton.addEventListener('click', addButtonBlock);
  btnAddHeading.addEventListener('click', addHeadingBlock);
  btnAddCols2.addEventListener('click', addCols2Block);

  loadBlocks();
});
</script>
</body>
</html>
