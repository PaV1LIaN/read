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
?>
<!doctype html>
<html lang="ru">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>SiteBuilder</title>

  <?php $APPLICATION->ShowHead(); ?>

  <style>
    html, body { height: 100%; margin: 0; }
    body { font-family: Arial, sans-serif; background:#f6f7f8; }
    .wrap { padding: 24px; }
    .box { background:#fff; border: 1px solid #e5e7ea; border-radius: 12px; padding: 16px; }
    table code { background: #f3f4f6; padding: 2px 6px; border-radius: 6px; }
    .muted { color:#6a737f; }

    /* ---- dialogs helpers ---- */
    .field { margin-top:10px; }
    .field label { display:block; font-size:12px; color:#6a737f; margin-bottom:4px; }
    .input, select, textarea { width:100%; padding:8px; border:1px solid #d0d7de; border-radius:8px; box-sizing:border-box; }

    .searchRow{display:flex; gap:10px; align-items:flex-end; flex-wrap:wrap; margin-bottom:10px;}
    .hint { font-size:12px; color:#6a737f; margin-top:8px; }

    /* ---- pages tree (better UI) ---- */
    .tree { margin-top: 12px; }
    .node { border:1px solid #eef0f2; border-radius:14px; padding:12px; background:#fff; margin-top:10px; }
    .nodeHead { display:flex; gap:12px; align-items:flex-start; justify-content:space-between; flex-wrap:wrap; }
    .nodeLeft { display:flex; gap:10px; align-items:flex-start; }
    .nodeIcon {
      width:22px; height:22px; border-radius:8px; background:#f3f4f6;
      display:flex; align-items:center; justify-content:center; color:#6a737f; font-weight:700;
      flex:0 0 auto;
    }
    .nodeMain { min-width: 240px; }
    .nodeTitleLine { display:flex; gap:8px; align-items:center; flex-wrap:wrap; }
    .nodeTitleLine b { font-size:14px; }
    .nodeSlug { font-size:12px; background:#f3f4f6; padding:2px 8px; border-radius:999px; color:#374151; }
    .nodeMeta { margin-top:4px; font-size:12px; color:#6a737f; display:flex; gap:10px; flex-wrap:wrap; }
    .nodeBtns { display:flex; gap:6px; flex-wrap:wrap; align-items:center; }
    .children { margin-left:18px; border-left:2px dashed #e5e7ea; padding-left:12px; margin-top:10px; }

    .btnTiny { padding:0 8px; height:26px; line-height:26px; }

    /* ---- picker cards (reused for parent picker) ---- */
    .secGrid{display:grid;gap:12px;margin-top:12px;}
    @media (min-width: 820px){ .secGrid{grid-template-columns: 1fr 1fr;} }
    .secCard{border:1px solid #e5e7ea;border-radius:14px;background:#fff;padding:12px;}
    .secTitle{font-weight:800;}
    .secMeta{color:#6a737f;font-size:12px;margin-top:4px;}
    .secSearch{margin-top:10px;}
  </style>
</head>
<body>
  <div class="wrap">
    <div class="box">
      <h1 style="margin-top:0;">SiteBuilder</h1>

      <div>Ты авторизован как: <b><?=htmlspecialcharsbx($USER->GetLogin())?></b></div>
      <div style="margin-top:10px;" class="muted">
        Открыто напрямую: <code>/local/sitebuilder/index.php</code>
      </div>

      <div style="margin-top:14px;">
        <button class="ui-btn ui-btn-primary" id="btnCreateSite">Создать сайт</button>
      </div>

      <div style="margin-top:14px;" id="sitesBox"></div>
    </div>
  </div>

<script>
BX.ready(function () {
  const sitesBox = document.getElementById('sitesBox');
  const btnCreate = document.getElementById('btnCreateSite');

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

  // ---------- SITES ----------
  function renderSites(sites) {
    if (!sites || !sites.length) {
      sitesBox.innerHTML = '<div class="muted">Сайтов, доступных тебе, пока нет. Создай первый.</div>';
      return;
    }

    const rows = sites.map(s => `
      <tr>
        <td style="padding:8px;border-bottom:1px solid #eee;">${s.id}</td>
        <td style="padding:8px;border-bottom:1px solid #eee;">${BX.util.htmlspecialchars(s.name)}</td>
        <td style="padding:8px;border-bottom:1px solid #eee;"><code>${BX.util.htmlspecialchars(s.slug)}</code></td>
        <td style="padding:8px;border-bottom:1px solid #eee;color:#6a737f;">${BX.util.htmlspecialchars(s.createdAt)}</td>
        <td style="padding:8px;border-bottom:1px solid #eee; white-space:nowrap;">
          <a class="ui-btn ui-btn-light ui-btn-xs"
             href="/local/sitebuilder/menu.php?siteId=${s.id}">Меню</a>

          <button class="ui-btn ui-btn-light ui-btn-xs"
                  data-open-pages-site-id="${s.id}"
                  data-open-pages-site-name="${BX.util.htmlspecialchars(s.name)}">Страницы</button>

          <a class="ui-btn ui-btn-light ui-btn-xs"
             href="/local/sitebuilder/files.php?siteId=${s.id}"
             target="_blank">Файлы</a>

          <button class="ui-btn ui-btn-light ui-btn-xs"
                  data-open-access-site-id="${s.id}"
                  data-open-access-site-name="${BX.util.htmlspecialchars(s.name)}">Доступы</button>

          <button class="ui-btn ui-btn-danger ui-btn-xs" data-delete-site-id="${s.id}">Удалить</button>
        </td>
      </tr>
    `).join('');

    sitesBox.innerHTML = `
      <table style="width:100%; border-collapse:collapse; margin-top:6px;">
        <thead>
          <tr>
            <th style="text-align:left;padding:8px;border-bottom:1px solid #eee;">ID</th>
            <th style="text-align:left;padding:8px;border-bottom:1px solid #eee;">Название</th>
            <th style="text-align:left;padding:8px;border-bottom:1px solid #eee;">Slug</th>
            <th style="text-align:left;padding:8px;border-bottom:1px solid #eee;">Создан</th>
            <th style="text-align:left;padding:8px;border-bottom:1px solid #eee;">Действия</th>
          </tr>
        </thead>
        <tbody>${rows}</tbody>
      </table>
    `;
  }

  function loadSites() {
    api('site.list').then(res => {
      if (!res || res.ok !== true) {
        notify('Не удалось загрузить список сайтов');
        return;
      }
      renderSites(res.sites);
    }).catch(() => notify('Ошибка запроса site.list'));
  }

  function openCreateSiteDialog() {
    const formHtml = `
      <div style="display:flex; flex-direction:column; gap:10px;">
        <div class="field">
          <label>Название сайта</label>
          <input type="text" id="sb_name" class="input" placeholder="Например: Лаборатория" />
        </div>
        <div class="field">
          <label>Slug (необязательно)</label>
          <input type="text" id="sb_slug" class="input" placeholder="lab (если пусто — сделаем автоматически)" />
        </div>
      </div>
    `;

    BX.UI.Dialogs.MessageBox.show({
      title: 'Создать сайт',
      message: formHtml,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mb) {
        const name = document.getElementById('sb_name')?.value?.trim() || '';
        const slug = document.getElementById('sb_slug')?.value?.trim() || '';

        if (!name) { notify('Введите название сайта'); return; }

        api('site.create', { name, slug }).then(res => {
          if (!res || res.ok !== true) { notify('Не удалось создать сайт'); return; }
          notify(`Сайт создан: ${BX.util.htmlspecialchars(res.site.name)} (${BX.util.htmlspecialchars(res.site.slug)})`);
          mb.close();
          loadSites();
        }).catch(() => notify('Ошибка запроса site.create'));
      }
    });
  }

  // ---------- ACCESS UI ----------
  function renderAccess(container, access, siteId) {
    if (!access || !access.length) {
      container.innerHTML = '<div class="muted">Правил доступа пока нет (кроме владельца).</div>';
      return;
    }

    const rows = access.map(r => {
      const code = r.accessCode || '';
      const role = r.role || '';
      const userId = (code.startsWith('U') ? parseInt(code.slice(1), 10) : 0);

      return `
        <tr>
          <td style="padding:8px;border-bottom:1px solid #eee;"><code>${BX.util.htmlspecialchars(code)}</code></td>
          <td style="padding:8px;border-bottom:1px solid #eee;">${BX.util.htmlspecialchars(role)}</td>
          <td style="padding:8px;border-bottom:1px solid #eee; white-space:nowrap;">
            <button class="ui-btn ui-btn-danger ui-btn-xs"
                    data-access-del-site-id="${siteId}"
                    data-access-del-user-id="${userId}">Удалить</button>
          </td>
        </tr>
      `;
    }).join('');

    container.innerHTML = `
      <table style="width:100%; border-collapse:collapse; margin-top:6px;">
        <thead>
          <tr>
            <th style="text-align:left;padding:8px;border-bottom:1px solid #eee;">AccessCode</th>
            <th style="text-align:left;padding:8px;border-bottom:1px solid #eee;">Role</th>
            <th style="text-align:left;padding:8px;border-bottom:1px solid #eee;">Действия</th>
          </tr>
        </thead>
        <tbody>${rows}</tbody>
      </table>
    `;
  }

  function loadAccess(siteId, container) {
    api('access.list', { siteId }).then(res => {
      if (!res || res.ok !== true) { notify('Нет прав на просмотр/изменение доступов (нужен OWNER)'); return; }
      renderAccess(container, res.access, siteId);
    }).catch(() => notify('Ошибка access.list'));
  }

  function openAccessDialog(siteId, siteName) {
    const html = `
      <div>
        <div class="muted" style="margin-bottom:10px;">
          Управление доступами (только OWNER). Пока добавляем по <b>UserID</b>.
        </div>

        <div style="display:flex; gap:8px; flex-wrap:wrap; align-items:flex-end; margin-bottom:10px;">
          <div style="flex:1; min-width:160px;">
            <div style="font-size:12px;color:#6a737f;margin-bottom:4px;">UserID</div>
            <input type="number" id="acc_user_id" class="input" placeholder="Например: 15" />
          </div>
          <div style="min-width:160px;">
            <div style="font-size:12px;color:#6a737f;margin-bottom:4px;">Role</div>
            <select id="acc_role" class="input">
              <option value="VIEWER">VIEWER</option>
              <option value="EDITOR">EDITOR</option>
              <option value="ADMIN">ADMIN</option>
              <option value="OWNER">OWNER</option>
            </select>
          </div>
          <div>
            <button class="ui-btn ui-btn-primary" id="btnAccSet">Назначить</button>
          </div>
        </div>

        <div id="accessBox"></div>
      </div>
    `;

    BX.UI.Dialogs.MessageBox.show({
      title: 'Доступы сайта: ' + BX.util.htmlspecialchars(siteName),
      message: html,
      buttons: BX.UI.Dialogs.MessageBoxButtons.CLOSE
    });

    setTimeout(function () {
      const box = document.getElementById('accessBox');
      if (!box) return;

      loadAccess(siteId, box);

      document.getElementById('btnAccSet')?.addEventListener('click', function () {
        const userId = parseInt(document.getElementById('acc_user_id')?.value || '0', 10);
        const role = document.getElementById('acc_role')?.value || 'VIEWER';
        if (!userId) { notify('Введите UserID'); return; }

        api('access.set', { siteId, userId, role }).then(res => {
          if (!res || res.ok !== true) { notify('Не удалось назначить доступ (нужен OWNER)'); return; }
          notify('Доступ назначен');
          loadAccess(siteId, box);
          loadSites();
        }).catch(() => notify('Ошибка access.set'));
      });
    }, 0);
  }

  // ---------- PAGES (NEW TREE UI) ----------
  function buildTree(pages) {
    const byId = {};
    pages.forEach(p => { byId[p.id] = Object.assign({ children: [] }, p); });

    const roots = [];
    pages.forEach(p => {
      const pid = parseInt(p.parentId || 0, 10) || 0;
      if (pid && byId[pid]) byId[pid].children.push(byId[p.id]);
      else roots.push(byId[p.id]);
    });

    const sortFn = (a,b) => (parseInt(a.sort||500,10) - parseInt(b.sort||500,10)) || (a.id - b.id);
    const sortRec = (arr) => { arr.sort(sortFn); arr.forEach(x => sortRec(x.children)); };
    sortRec(roots);

    return { roots, byId };
  }

  function renderPagesTree(container, siteId, pages, q) {
    const query = (q || '').trim().toLowerCase();
    const matches = (p) => {
      if (!query) return true;
      const t = (p.title||'').toLowerCase();
      const s = (p.slug||'').toLowerCase();
      return t.includes(query) || s.includes(query) || String(p.id).includes(query);
    };

    const { roots } = buildTree(pages);

    const renderNode = (node) => {
      const kidsHtml = (node.children || [])
        .map(renderNode)
        .filter(Boolean)
        .join('');

      const selfMatch = matches(node);
      const hasVisibleKids = kidsHtml !== '';
      if (!selfMatch && !hasVisibleKids) return '';

      const pid = parseInt(node.parentId||0,10)||0;
      const parentLabel = pid ? `parent: #${pid}` : 'root';

      return `
        <div class="node">
          <div class="nodeHead">
            <div class="nodeLeft">
              <div class="nodeIcon">≡</div>
              <div class="nodeMain">
                <div class="nodeTitleLine">
                  <b>#${node.id} ${BX.util.htmlspecialchars(node.title || '')}</b>
                  <span class="nodeSlug">${BX.util.htmlspecialchars(node.slug || '')}</span>
                </div>
                <div class="nodeMeta">
                  <span>sort: ${parseInt(node.sort||500,10)}</span>
                  <span>${parentLabel}</span>
                </div>
              </div>
            </div>

            <div class="nodeBtns">
              <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-move="${node.id}" data-dir="up">↑</button>
              <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-move="${node.id}" data-dir="down">↓</button>

              <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-parent="${node.id}">Вложить…</button>
              <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-root="${node.id}">В корень</button>

              <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-rename="${node.id}">Имя/slug</button>

              <a class="ui-btn ui-btn-primary ui-btn-xs btnTiny"
                 href="/local/sitebuilder/editor.php?siteId=${siteId}&pageId=${node.id}"
                 target="_blank">Редактор</a>

              <a class="ui-btn ui-btn-light ui-btn-xs btnTiny"
                 href="/local/sitebuilder/view.php?siteId=${siteId}&pageId=${node.id}"
                 target="_blank">Открыть</a>

              <button class="ui-btn ui-btn-danger ui-btn-xs btnTiny" data-page-delete="${node.id}">Удалить</button>
            </div>
          </div>

          ${hasVisibleKids ? `<div class="children">${kidsHtml}</div>` : ''}
        </div>
      `;
    };

    const html = roots.map(renderNode).filter(Boolean).join('');
    container.innerHTML = html || '<div class="muted">Страниц пока нет.</div>';
  }

  function openCreatePageDialog(siteId, onDone) {
    const formHtml = `
      <div style="display:flex; flex-direction:column; gap:10px;">
        <div class="field">
          <label>Заголовок страницы</label>
          <input type="text" id="pg_title" class="input" placeholder="Например: Главная" />
        </div>
        <div class="field">
          <label>Slug (необязательно)</label>
          <input type="text" id="pg_slug" class="input" placeholder="home (если пусто — сделаем автоматически)" />
        </div>
      </div>
    `;

    BX.UI.Dialogs.MessageBox.show({
      title: 'Создать страницу',
      message: formHtml,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mb) {
        const title = document.getElementById('pg_title')?.value?.trim() || '';
        const slug  = document.getElementById('pg_slug')?.value?.trim() || '';
        if (!title) { notify('Введите заголовок страницы'); return; }

        api('page.create', { siteId, title, slug }).then(res => {
          if (!res || res.ok !== true) { notify('Не удалось создать страницу (возможно нет прав)'); return; }
          notify(`Страница создана: ${BX.util.htmlspecialchars(res.page.title)} (${BX.util.htmlspecialchars(res.page.slug)})`);
          mb.close();
          if (typeof onDone === 'function') onDone();
        }).catch(() => notify('Ошибка page.create'));
      }
    });
  }

  function openRenamePageDialog(siteId, pageId, pagesCache, reload) {
    const cur = (pagesCache || []).find(x => parseInt(x.id,10) === parseInt(pageId,10));
    const curTitle = cur?.title || '';
    const curSlug  = cur?.slug || '';

    BX.UI.Dialogs.MessageBox.show({
      title: 'Имя / slug для #' + pageId,
      message: `
        <div style="display:flex; flex-direction:column; gap:10px;">
          <div class="field">
            <label>Заголовок</label>
            <input id="rn_title" class="input" value="${BX.util.htmlspecialchars(curTitle)}" />
          </div>
          <div class="field">
            <label>Slug (можно пусто — пересчитаем)</label>
            <input id="rn_slug" class="input" value="${BX.util.htmlspecialchars(curSlug)}" />
          </div>
          <div class="hint">Slug должен быть уникален в пределах сайта — если совпадёт, добавим суффикс.</div>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function(mb){
        const title = (document.getElementById('rn_title')?.value || '').trim();
        const slug = (document.getElementById('rn_slug')?.value || '').trim();
        if (!title) { notify('Заголовок не может быть пустым'); return; }

        api('page.updateMeta', { id: pageId, title, slug }).then(r => {
          if (!r || r.ok !== true) { notify('Не удалось сохранить'); return; }
          notify('Сохранено');
          mb.close();
          reload();
        }).catch(()=>notify('Ошибка page.updateMeta'));
      }
    });
  }

  function openSetParentDialog(siteId, pageId, pagesCache, reload) {
    const pages = Array.isArray(pagesCache) ? pagesCache : [];
    const current = pages.find(x => parseInt(x.id,10) === parseInt(pageId,10));
    const currentParentId = parseInt(current?.parentId || 0, 10) || 0;

    // ВАЖНО: выбранный родитель должен быть изменяемым, а не константой
    let selectedParentId = currentParentId;

    const { roots } = buildTree(pages);
    const flat = [];
    const walk = (arr, depth) => {
        arr.forEach(n => {
        flat.push({ id: n.id, title: n.title||'', slug:n.slug||'', depth });
        walk(n.children||[], depth+1);
        });
    };
    walk(roots, 0);

    const renderListInner = (q) => {
        const query = (q||'').trim().toLowerCase();

        const items = flat.filter(x => {
        if (parseInt(x.id,10) === parseInt(pageId,10)) return false; // нельзя выбрать себя
        if (!query) return true;
        return (x.title||'').toLowerCase().includes(query)
            || (x.slug||'').toLowerCase().includes(query)
            || String(x.id).includes(query);
        });

        const rows = items.map(x => {
        const pad = 12 + x.depth * 16;
        const active = (parseInt(x.id,10) === selectedParentId)
            ? 'background:#eff6ff;border-color:#bfdbfe;'
            : '';
        return `
            <div class="secCard" data-parent-pick="${x.id}"
                style="cursor:pointer; padding-left:${pad}px; ${active}">
            <div class="secTitle">#${x.id} ${BX.util.htmlspecialchars(x.title)}</div>
            <div class="secMeta">${BX.util.htmlspecialchars(x.slug)}</div>
            </div>
        `;
        }).join('');

        const activeRoot = selectedParentId === 0 ? 'background:#eff6ff;border-color:#bfdbfe;' : '';

        return `
        <div class="secSearch">
            <input id="par_q" class="input" placeholder="Поиск родителя..." value="${BX.util.htmlspecialchars(q||'')}">
        </div>

        <div class="secGrid" style="grid-template-columns: 1fr; margin-top:10px;">
            <div class="secCard" data-parent-pick="0" style="cursor:pointer; ${activeRoot}">
            <div class="secTitle">— Корень —</div>
            <div class="secMeta">Сделать страницу верхнего уровня</div>
            </div>

            ${rows || '<div class="muted">Ничего не найдено</div>'}
        </div>

        <div class="hint" style="margin-top:10px;">Кликни по карточке родителя. Потом нажми OK.</div>
        `;
    };

    BX.UI.Dialogs.MessageBox.show({
        title: 'Вложить страницу #' + pageId,
        message: `<div id="sb_parent_picker_root">${renderListInner('')}</div>`,
        buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
        onOk: function(mb){
        const parentId = parseInt(selectedParentId || 0, 10) || 0;

        api('page.setParent', { id: pageId, parentId }).then(r=>{
            if (!r || r.ok !== true) { notify('Не удалось изменить parent'); return; }
            notify('Готово');
            mb.close();
            reload();
        }).catch(()=>notify('Ошибка page.setParent'));
        }
    });

    setTimeout(() => {
        const root = document.getElementById('sb_parent_picker_root');
        if (!root) return;

        const bind = () => {
        const q = document.getElementById('par_q');
        if (q) {
            q.oninput = () => {
            root.innerHTML = renderListInner(q.value);
            bind();
            };
        }

        root.querySelectorAll('[data-parent-pick]').forEach(el => {
            el.onclick = () => {
            const id = parseInt(el.getAttribute('data-parent-pick')||'0',10) || 0;
            selectedParentId = id; // ✅ сохраняем выбор
            const curQ = document.getElementById('par_q')?.value || '';
            root.innerHTML = renderListInner(curQ); // ✅ перерисовка уже с новым selectedParentId
            bind();
            };
        });
        };

        bind();
    }, 0);
  }
  async function openPagesDialog(siteId, siteName) {
    let pagesCache = [];

    const html = `
      <div>
        <div class="searchRow">
          <div>
            <button class="ui-btn ui-btn-primary ui-btn-xs" id="btnCreatePage">Создать страницу</button>
          </div>
          <div class="field" style="flex:1; min-width:220px;">
            <label>Поиск</label>
            <input id="pg_q" class="input" placeholder="title / slug / id..." />
          </div>
        </div>

        <div class="hint">
          Дерево: <code>parentId</code>. Порядок: <code>sort</code> (стрелки ↑/↓ меняют порядок среди “соседей”).
        </div>

        <div id="pagesBox" class="tree"></div>
      </div>
    `;

    BX.UI.Dialogs.MessageBox.show({
      title: 'Страницы сайта: ' + BX.util.htmlspecialchars(siteName),
      message: html,
      buttons: BX.UI.Dialogs.MessageBoxButtons.CLOSE
    });

    const loadAndRender = async () => {
      const container = document.getElementById('pagesBox');
      const q = document.getElementById('pg_q')?.value || '';
      if (!container) return;

      try {
        const res = await api('page.list', { siteId });
        if (!res || res.ok !== true) { notify('Не удалось загрузить страницы (возможно нет прав)'); return; }
        pagesCache = res.pages || [];
        renderPagesTree(container, siteId, pagesCache, q);
      } catch (e) {
        notify('Ошибка page.list');
      }
    };

    setTimeout(() => {
      const container = document.getElementById('pagesBox');
      const q = document.getElementById('pg_q');
      const btn = document.getElementById('btnCreatePage');

      if (btn) btn.onclick = () => openCreatePageDialog(siteId, loadAndRender);
      if (q) q.oninput = () => loadAndRender();

      if (container) {
        container.addEventListener('click', function(e){
          const mv = e.target.closest('[data-page-move]');
          if (mv) {
            const id = parseInt(mv.getAttribute('data-page-move'),10);
            const dir = mv.getAttribute('data-dir');
            api('page.move', { id, dir }).then(r=>{
              if(!r || r.ok!==true){ notify('Не удалось переместить'); return; }
              loadAndRender();
            }).catch(()=>notify('Ошибка page.move'));
            return;
          }

          const rn = e.target.closest('[data-page-rename]');
          if (rn) {
            const id = parseInt(rn.getAttribute('data-page-rename'),10);
            openRenamePageDialog(siteId, id, pagesCache, loadAndRender);
            return;
          }

          const pr = e.target.closest('[data-page-parent]');
          if (pr) {
            const id = parseInt(pr.getAttribute('data-page-parent'),10);
            openSetParentDialog(siteId, id, pagesCache, loadAndRender);
            return;
          }

          const rt = e.target.closest('[data-page-root]');
          if (rt) {
            const id = parseInt(rt.getAttribute('data-page-root'),10);
            api('page.setParent', { id, parentId: 0 }).then(r=>{
              if(!r || r.ok!==true){ notify('Не удалось'); return; }
              loadAndRender();
            }).catch(()=>notify('Ошибка page.setParent'));
            return;
          }

          const del = e.target.closest('[data-page-delete]');
          if (del) {
            const id = parseInt(del.getAttribute('data-page-delete'),10);
            BX.UI.Dialogs.MessageBox.show({
              title: 'Удалить страницу #' + id + '?',
              message: 'Нужны права EDITOR/ADMIN/OWNER. Продолжить?',
              buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
              onOk: function(mb){
                api('page.delete', { id }).then(r=>{
                  if(!r || r.ok!==true){ notify('Не удалось удалить (возможно нет прав)'); return; }
                  notify('Страница удалена');
                  mb.close();
                  loadAndRender();
                }).catch(()=>notify('Ошибка page.delete'));
              }
            });
            return;
          }
        });
      }

      loadAndRender();
    }, 0);
  }

  // ---------- EVENTS ----------
  document.addEventListener('click', function (e) {
    const delSiteBtn = e.target.closest('[data-delete-site-id]');
    if (delSiteBtn) {
      const id = parseInt(delSiteBtn.getAttribute('data-delete-site-id'), 10);
      if (!id) return;

      BX.UI.Dialogs.MessageBox.show({
        title: 'Удалить сайт?',
        message: 'Удаление доступно только владельцу (OWNER). Продолжить?',
        buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
        onOk: function (mb) {
          api('site.delete', { id }).then(res => {
            if (!res || res.ok !== true) { notify('Не удалось удалить сайт (нужен OWNER)'); return; }
            notify('Сайт удалён');
            mb.close();
            loadSites();
          }).catch(() => notify('Ошибка site.delete'));
        }
      });
      return;
    }

    const openPagesBtn = e.target.closest('[data-open-pages-site-id]');
    if (openPagesBtn) {
      const siteId = parseInt(openPagesBtn.getAttribute('data-open-pages-site-id'), 10);
      const siteName = openPagesBtn.getAttribute('data-open-pages-site-name') || ('ID ' + siteId);
      if (!siteId) return;
      openPagesDialog(siteId, siteName);
      return;
    }

    const openAccBtn = e.target.closest('[data-open-access-site-id]');
    if (openAccBtn) {
      const siteId = parseInt(openAccBtn.getAttribute('data-open-access-site-id'), 10);
      const siteName = openAccBtn.getAttribute('data-open-access-site-name') || ('ID ' + siteId);
      if (!siteId) return;
      openAccessDialog(siteId, siteName);
      return;
    }

    const delAccBtn = e.target.closest('[data-access-del-site-id]');
    if (delAccBtn) {
      const siteId = parseInt(delAccBtn.getAttribute('data-access-del-site-id'), 10);
      const userId = parseInt(delAccBtn.getAttribute('data-access-del-user-id'), 10);
      if (!siteId || !userId) return;

      BX.UI.Dialogs.MessageBox.show({
        title: 'Удалить правило доступа?',
        message: 'Продолжить? (только OWNER)',
        buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
        onOk: function (mb) {
          api('access.delete', { siteId, userId }).then(res => {
            if (!res || res.ok !== true) { notify('Не удалось удалить правило (нужен OWNER)'); return; }
            notify('Удалено');
            mb.close();

            const accessBox = document.getElementById('accessBox');
            if (accessBox) loadAccess(siteId, accessBox);
            loadSites();
          }).catch(() => notify('Ошибка access.delete'));
        }
      });
      return;
    }
  });

  if (btnCreate) btnCreate.addEventListener('click', openCreateSiteDialog);

  loadSites();
});
</script>
</body>
</html>
