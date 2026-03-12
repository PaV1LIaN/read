Идём дальше. Ниже — полный готовый layout.php для редактирования зон сайта:

Header

Footer

Left

Right


Что умеет:

загрузить layout settings,

переключать зоны,

показывать список блоков выбранной зоны,

добавлять блоки,

редактировать блоки,

удалять,

двигать вверх/вниз,

менять layout settings сайта.


Сразу использует твои layout.* action'ы из api.php.

Создай файл:

/local/sitebuilder/layout.php

и вставь туда:

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
?>
<!doctype html>
<html lang="ru">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Layout сайта</title>
  <?php $APPLICATION->ShowHead(); ?>
  <style>
    body { font-family: Arial, sans-serif; margin:0; background:#f6f7f8; color:#111; }
    .top {
      position: sticky;
      top: 0;
      z-index: 50;
      background: rgba(255,255,255,.94);
      backdrop-filter: blur(10px);
      border-bottom:1px solid #e5e7ea;
      padding:12px 16px;
      display:flex;
      gap:10px;
      align-items:center;
      flex-wrap:wrap;
    }
    .content { padding:18px; }
    .card { background:#fff; border:1px solid #e5e7ea; border-radius:14px; padding:16px; }
    .muted { color:#6a737f; }
    .row { display:flex; gap:10px; align-items:center; justify-content:space-between; flex-wrap:wrap; }
    .btns { display:flex; gap:6px; flex-wrap:wrap; }
    .field { margin-top:12px; }
    .field label { display:block; font-size:12px; color:#6a737f; margin-bottom:4px; }
    .input, select, textarea {
      width:100%;
      padding:8px;
      border:1px solid #d0d7de;
      border-radius:8px;
      box-sizing:border-box;
      background:#fff;
    }
    code { background:#f3f4f6; padding:2px 6px; border-radius:6px; }
    pre { white-space:pre-wrap; margin:10px 0 0; background:#f9fafb; border:1px solid #eee; border-radius:8px; padding:10px; }

    .zones {
      display:flex;
      gap:8px;
      flex-wrap:wrap;
      margin-top:12px;
    }

    .zoneBtn.active {
      background:#2563eb !important;
      border-color:#2563eb !important;
      color:#fff !important;
    }

    .layoutGrid {
      display:grid;
      grid-template-columns: 360px 1fr;
      gap:16px;
      align-items:start;
      margin-top:16px;
    }
    @media (max-width: 980px) {
      .layoutGrid { grid-template-columns: 1fr; }
    }

    .block {
      border:1px solid #e5e7ea;
      border-radius:14px;
      padding:12px;
      margin-top:12px;
      background:#fff;
      box-shadow:0 1px 2px rgba(0,0,0,.03);
    }

    .blockHeader {
      display:flex;
      gap:10px;
      align-items:flex-start;
      justify-content:space-between;
      flex-wrap:wrap;
    }

    .blockLeft {
      display:flex;
      flex-direction:column;
      gap:6px;
      min-width:220px;
    }

    .blockTitleRow {
      display:flex;
      gap:8px;
      align-items:center;
      flex-wrap:wrap;
    }

    .blockTypeBadge {
      display:inline-flex;
      align-items:center;
      padding:3px 8px;
      border-radius:999px;
      font-size:11px;
      font-weight:700;
      border:1px solid #e5e7ea;
      background:#f8fafc;
      color:#334155;
    }

    .blockMeta {
      display:flex;
      gap:8px;
      flex-wrap:wrap;
      color:#6a737f;
      font-size:12px;
    }

    .imgPrev {
      margin-top:10px;
      max-width:420px;
      border:1px solid #eee;
      border-radius:10px;
      overflow:hidden;
      background:#fafafa;
    }
    .imgPrev img { display:block; width:100%; height:auto; }

    .headingPreview {
      margin-top:10px;
      border:1px dashed #e5e7ea;
      border-radius:10px;
      padding:10px;
    }
    .headingPreview h1, .headingPreview h2, .headingPreview h3 { margin:0; }

    .colsPreview {
      margin-top:10px;
      border:1px dashed #e5e7ea;
      border-radius:10px;
      padding:10px;
      display:grid;
      gap:10px;
    }
    .colsPreview .cell {
      background:#fafafa;
      border:1px solid #eee;
      border-radius:10px;
      padding:10px;
      min-height:48px;
    }
    .colsPreview pre { margin:0; background:transparent; border:none; padding:0; }

    .galPrev { margin-top:10px; display:grid; gap:10px; }
    .galPrev img {
      width:100%;
      height:auto;
      display:block;
      border-radius:10px;
      border:1px solid #eee;
      background:#fafafa;
    }

    .cardsBuilder { margin-top:10px; }
    .cardsBuilder .item { border:1px solid #e5e7ea; border-radius:12px; padding:10px; margin-top:10px; background:#fff; }
    .cardsBuilder .itemHead { display:flex; gap:8px; align-items:center; justify-content:space-between; flex-wrap:wrap; }
    .cardsBuilder .miniBtns { display:flex; gap:6px; flex-wrap:wrap; }
    .cardsBuilder .grid2 { display:grid; grid-template-columns:1fr 1fr; gap:10px; }
    @media (max-width:720px){ .cardsBuilder .grid2 { grid-template-columns:1fr; } }

    .block[data-type="text"] .blockTypeBadge { background:#eff6ff; color:#1d4ed8; border-color:#bfdbfe; }
    .block[data-type="image"] .blockTypeBadge { background:#f5f3ff; color:#6d28d9; border-color:#ddd6fe; }
    .block[data-type="button"] .blockTypeBadge { background:#ecfeff; color:#0f766e; border-color:#a5f3fc; }
    .block[data-type="heading"] .blockTypeBadge { background:#fef3c7; color:#92400e; border-color:#fcd34d; }
    .block[data-type="columns2"] .blockTypeBadge { background:#f3f4f6; color:#374151; border-color:#d1d5db; }
    .block[data-type="gallery"] .blockTypeBadge { background:#fdf2f8; color:#be185d; border-color:#fbcfe8; }
    .block[data-type="spacer"] .blockTypeBadge { background:#f8fafc; color:#475569; border-color:#cbd5e1; }
    .block[data-type="card"] .blockTypeBadge { background:#eef2ff; color:#3730a3; border-color:#c7d2fe; }
    .block[data-type="cards"] .blockTypeBadge { background:#ecfccb; color:#3f6212; border-color:#bef264; }
  </style>
</head>
<body>
  <div class="top">
    <a href="/local/sitebuilder/index.php">← Назад</a>
    <div class="muted">Layout сайта</div>
    <div class="muted">|</div>
    <div><b>siteId:</b> <code><?= (int)$siteId ?></code></div>
  </div>

  <div class="content">
    <div class="layoutGrid">
      <div class="card">
        <div class="row">
          <div><b>Настройки layout</b></div>
        </div>

        <div class="field">
          <label><input type="checkbox" id="showHeader"> Показывать header</label>
        </div>
        <div class="field">
          <label><input type="checkbox" id="showFooter"> Показывать footer</label>
        </div>
        <div class="field">
          <label><input type="checkbox" id="showLeft"> Показывать left</label>
        </div>
        <div class="field">
          <label><input type="checkbox" id="showRight"> Показывать right</label>
        </div>
        <div class="field">
          <label>Ширина left</label>
          <input class="input" type="number" id="leftWidth" min="160" max="500" value="260">
        </div>
        <div class="field">
          <label>Ширина right</label>
          <input class="input" type="number" id="rightWidth" min="160" max="500" value="260">
        </div>

        <div class="btns" style="margin-top:14px;">
          <button class="ui-btn ui-btn-primary" id="btnSaveLayoutSettings">Сохранить настройки</button>
        </div>

        <div class="zones">
          <button class="ui-btn ui-btn-light zoneBtn active" data-zone="header">Header</button>
          <button class="ui-btn ui-btn-light zoneBtn" data-zone="footer">Footer</button>
          <button class="ui-btn ui-btn-light zoneBtn" data-zone="left">Left</button>
          <button class="ui-btn ui-btn-light zoneBtn" data-zone="right">Right</button>
        </div>

        <div class="muted" style="margin-top:12px;">
          Выбрана зона: <b id="zoneLabel">header</b>
        </div>

        <div class="btns" style="margin-top:12px;">
          <button class="ui-btn ui-btn-primary" id="btnAddText">+ Text</button>
          <button class="ui-btn ui-btn-primary" id="btnAddImage">+ Image</button>
          <button class="ui-btn ui-btn-primary" id="btnAddButton">+ Button</button>
          <button class="ui-btn ui-btn-primary" id="btnAddHeading">+ Heading</button>
          <button class="ui-btn ui-btn-primary" id="btnAddCols2">+ Columns2</button>
          <button class="ui-btn ui-btn-primary" id="btnAddGallery">+ Gallery</button>
          <button class="ui-btn ui-btn-primary" id="btnAddSpacer">+ Spacer</button>
          <button class="ui-btn ui-btn-primary" id="btnAddCard">+ Card</button>
          <button class="ui-btn ui-btn-primary" id="btnAddCards">+ Cards</button>
        </div>
      </div>

      <div class="card">
        <div class="row">
          <div><b>Блоки зоны</b></div>
          <div class="muted" id="blocksMeta"></div>
        </div>
        <div id="blocksBox" style="margin-top:12px;"></div>
      </div>
    </div>
  </div>

<script>
BX.ready(function () {
  const siteId = <?= (int)$siteId ?>;
  const blocksBox = document.getElementById('blocksBox');
  const zoneLabel = document.getElementById('zoneLabel');
  const blocksMeta = document.getElementById('blocksMeta');

  let currentZone = 'header';
  let layoutSettings = {
    showHeader: true,
    showFooter: true,
    showLeft: false,
    showRight: false,
    leftWidth: 260,
    rightWidth: 260
  };

  function notify(msg) {
    BX.UI.Notification.Center.notify({ content: msg });
  }

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
    return (variant === 'secondary') ? 'ui-btn-light-border' : 'ui-btn-primary';
  }

  function headingTag(level) {
    return (level === 'h1' || level === 'h2' || level === 'h3') ? level : 'h2';
  }

  function headingAlign(align) {
    return (align === 'left' || align === 'center' || align === 'right') ? align : 'left';
  }

  function colsGridTemplate(ratio) {
    if (ratio === '33-67') return '1fr 2fr';
    if (ratio === '67-33') return '2fr 1fr';
    return '1fr 1fr';
  }

  function galleryTemplate(columns) {
    if (columns === 2) return '1fr 1fr';
    if (columns === 4) return '1fr 1fr 1fr 1fr';
    return '1fr 1fr 1fr';
  }

  function fillSettingsForm() {
    document.getElementById('showHeader').checked = !!layoutSettings.showHeader;
    document.getElementById('showFooter').checked = !!layoutSettings.showFooter;
    document.getElementById('showLeft').checked = !!layoutSettings.showLeft;
    document.getElementById('showRight').checked = !!layoutSettings.showRight;
    document.getElementById('leftWidth').value = parseInt(layoutSettings.leftWidth || 260, 10);
    document.getElementById('rightWidth').value = parseInt(layoutSettings.rightWidth || 260, 10);
  }

  function buildBlockShell(id, type, sort, bodyHtml, buttonsHtml, extraMetaHtml = '') {
    return `
      <div class="block" data-type="${BX.util.htmlspecialchars(type)}" data-block-id="${id}">
        <div class="blockHeader">
          <div class="blockLeft">
            <div class="blockTitleRow">
              <b>#${id}</b>
              <span class="blockTypeBadge">${BX.util.htmlspecialchars(type)}</span>
            </div>
            <div class="blockMeta">
              <span>sort: ${sort}</span>
              ${extraMetaHtml}
            </div>
          </div>
          <div class="btns">
            ${buttonsHtml}
          </div>
        </div>
        <div style="margin-top:10px;">
          ${bodyHtml}
        </div>
      </div>
    `;
  }

  function renderBlocks(blocks) {
    blocksMeta.textContent = `Зона: ${currentZone} • блоков: ${blocks.length}`;

    if (!blocks.length) {
      blocksBox.innerHTML = '<div class="muted">В этой зоне блоков пока нет.</div>';
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
        return buildBlockShell(
          id, type, sort,
          `<pre>${BX.util.htmlspecialchars(text)}</pre>`,
          `${commonBtns}
           <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-text-id="${id}">Редактировать</button>
           <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>`
        );
      }

      if (type === 'image') {
        const fileId = b.content && b.content.fileId ? parseInt(b.content.fileId, 10) : 0;
        const alt = b.content && typeof b.content.alt === 'string' ? b.content.alt : '';
        const img = fileId
          ? `<div class="imgPrev"><img src="${fileDownloadUrl(fileId)}" alt="${BX.util.htmlspecialchars(alt)}"></div>`
          : '<div class="muted">Файл не выбран</div>';

        return buildBlockShell(
          id, type, sort,
          `<div class="muted">alt: ${BX.util.htmlspecialchars(alt)}</div>${img}`,
          `${commonBtns}
           <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-image-id="${id}">Редактировать</button>
           <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>`,
          `<span>fileId: ${fileId || '-'}</span>`
        );
      }

      if (type === 'button') {
        const text = (b.content && typeof b.content.text === 'string') ? b.content.text : '';
        const url = (b.content && typeof b.content.url === 'string') ? b.content.url : '';
        const variant = (b.content && typeof b.content.variant === 'string') ? b.content.variant : 'primary';

        return buildBlockShell(
          id, type, sort,
          `<div class="muted">url: ${BX.util.htmlspecialchars(url)}</div>
           <a class="ui-btn ${variant === 'secondary' ? 'ui-btn-light-border' : 'ui-btn-primary'}" href="${BX.util.htmlspecialchars(url)}" target="_blank" rel="noopener noreferrer">${BX.util.htmlspecialchars(text)}</a>`,
          `${commonBtns}
           <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-button-id="${id}">Редактировать</button>
           <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>`,
          `<span>variant: ${BX.util.htmlspecialchars(variant)}</span>`
        );
      }

      if (type === 'heading') {
        const text = (b.content && typeof b.content.text === 'string') ? b.content.text : '';
        const level = (b.content && typeof b.content.level === 'string') ? b.content.level : 'h2';
        const align = (b.content && typeof b.content.align === 'string') ? b.content.align : 'left';
        const tag = headingTag(level);
        const al = headingAlign(align);

        return buildBlockShell(
          id, type, sort,
          `<div class="headingPreview" style="text-align:${BX.util.htmlspecialchars(al)};">
             <${tag}>${BX.util.htmlspecialchars(text)}</${tag}>
           </div>`,
          `${commonBtns}
           <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-heading-id="${id}">Редактировать</button>
           <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>`,
          `<span>${BX.util.htmlspecialchars(tag)}</span><span>${BX.util.htmlspecialchars(al)}</span>`
        );
      }

      if (type === 'columns2') {
        const left = (b.content && typeof b.content.left === 'string') ? b.content.left : '';
        const right = (b.content && typeof b.content.right === 'string') ? b.content.right : '';
        const ratio = (b.content && typeof b.content.ratio === 'string') ? b.content.ratio : '50-50';
        const tpl = colsGridTemplate(ratio);

        return buildBlockShell(
          id, type, sort,
          `<div class="colsPreview" style="grid-template-columns:${tpl};">
             <div class="cell"><pre>${BX.util.htmlspecialchars(left)}</pre></div>
             <div class="cell"><pre>${BX.util.htmlspecialchars(right)}</pre></div>
           </div>`,
          `${commonBtns}
           <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-cols2-id="${id}">Редактировать</button>
           <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>`,
          `<span>ratio: ${BX.util.htmlspecialchars(ratio)}</span>`
        );
      }

      if (type === 'gallery') {
        const columns = (b.content && b.content.columns) ? parseInt(b.content.columns, 10) : 3;
        const imgs = (b.content && Array.isArray(b.content.images)) ? b.content.images : [];
        const tpl = galleryTemplate(columns);

        const prev = imgs.map(it => {
          const fid = parseInt(it.fileId || 0, 10);
          if (!fid) return '';
          return `<img src="${fileDownloadUrl(fid)}" alt="${BX.util.htmlspecialchars(it.alt || '')}">`;
        }).join('');

        return buildBlockShell(
          id, type, sort,
          `<div class="galPrev" style="grid-template-columns:${tpl};">${prev}</div>`,
          `${commonBtns}
           <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-gallery-id="${id}">Редактировать</button>
           <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>`,
          `<span>cols: ${columns}</span><span>images: ${imgs.length}</span>`
        );
      }

      if (type === 'spacer') {
        const height = (b.content && b.content.height) ? parseInt(b.content.height, 10) : 40;
        const line = (b.content && (b.content.line === true || b.content.line === 'true')) ? true : false;

        return buildBlockShell(
          id, type, sort,
          `<div style="border:1px dashed #e5e7ea; border-radius:10px; padding:10px;">
             <div style="height:${height}px; position:relative; background:#fafafa; border-radius:10px;">
               ${line ? '<div style="position:absolute; left:0; right:0; top:50%; height:1px; background:#e5e7ea;"></div>' : ''}
             </div>
           </div>`,
          `${commonBtns}
           <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-spacer-id="${id}">Редактировать</button>
           <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>`,
          `<span>${height}px</span><span>line: ${line ? 'yes' : 'no'}</span>`
        );
      }

      if (type === 'card') {
        const title = (b.content && typeof b.content.title === 'string') ? b.content.title : '';
        const text = (b.content && typeof b.content.text === 'string') ? b.content.text : '';
        const imageFileId = (b.content && b.content.imageFileId) ? parseInt(b.content.imageFileId, 10) : 0;
        const buttonText = (b.content && typeof b.content.buttonText === 'string') ? b.content.buttonText : '';
        const buttonUrl = (b.content && typeo