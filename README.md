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
        BX.UI.Notification.Center.notify({ content: 'Не удалось загрузить список сайтов' });
        return;
      }
      renderSites(res.sites);
    }).catch(() => {
      BX.UI.Notification.Center.notify({ content: 'Ошибка запроса site.list' });
    });
  }

  function openCreateSiteDialog() {
    const formHtml = `
      <div style="display:flex; flex-direction:column; gap:10px;">
        <div>
          <div style="font-size:12px;color:#6a737f;margin-bottom:4px;">Название сайта</div>
          <input type="text" id="sb_name"
            style="width:100%; padding:8px; border:1px solid #d0d7de; border-radius:8px;"
            placeholder="Например: Лаборатория" />
        </div>
        <div>
          <div style="font-size:12px;color:#6a737f;margin-bottom:4px;">Slug (необязательно)</div>
          <input type="text" id="sb_slug"
            style="width:100%; padding:8px; border:1px solid #d0d7de; border-radius:8px;"
            placeholder="lab (если пусто — сделаем автоматически)" />
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

        if (!name) {
          BX.UI.Notification.Center.notify({ content: 'Введите название сайта' });
          return;
        }

        api('site.create', { name, slug }).then(res => {
          if (!res || res.ok !== true) {
            BX.UI.Notification.Center.notify({ content: 'Не удалось создать сайт' });
            return;
          }
          BX.UI.Notification.Center.notify({
            content: `Сайт создан: ${BX.util.htmlspecialchars(res.site.name)} (${BX.util.htmlspecialchars(res.site.slug)})`
          });
          mb.close();
          loadSites();
        }).catch(() => {
          BX.UI.Notification.Center.notify({ content: 'Ошибка запроса site.create' });
        });
      }
    });
  }

  // ---------- PAGES ----------
  function renderPages(container, pages) {
    if (!pages || !pages.length) {
      container.innerHTML = '<div class="muted">Страниц пока нет.</div>';
      return;
    }

    const rows = pages.map(p => `
      <tr>
        <td style="padding:8px;border-bottom:1px solid #eee;">${p.id}</td>
        <td style="padding:8px;border-bottom:1px solid #eee;">${BX.util.htmlspecialchars(p.title)}</td>
        <td style="padding:8px;border-bottom:1px solid #eee;"><code>${BX.util.htmlspecialchars(p.slug)}</code></td>
        <td style="padding:8px;border-bottom:1px solid #eee; white-space:nowrap;">
          <a class="ui-btn ui-btn-primary ui-btn-xs"
             href="/local/sitebuilder/editor.php?siteId=${p.siteId}&pageId=${p.id}"
             target="_blank">Редактор</a>
          <a class="ui-btn ui-btn-light ui-btn-xs"
             href="/local/sitebuilder/view.php?siteId=${p.siteId}&pageId=${p.id}"
             target="_blank">Открыть</a>
          <button class="ui-btn ui-btn-danger ui-btn-xs"
                  data-delete-page-id="${p.id}"
                  data-delete-page-site-id="${p.siteId}">Удалить</button>
        </td>
      </tr>
    `).join('');

    container.innerHTML = `
      <table style="width:100%; border-collapse:collapse; margin-top:6px;">
        <thead>
          <tr>
            <th style="text-align:left;padding:8px;border-bottom:1px solid #eee;">ID</th>
            <th style="text-align:left;padding:8px;border-bottom:1px solid #eee;">Заголовок</th>
            <th style="text-align:left;padding:8px;border-bottom:1px solid #eee;">Slug</th>
            <th style="text-align:left;padding:8px;border-bottom:1px solid #eee;">Действия</th>
          </tr>
        </thead>
        <tbody>${rows}</tbody>
      </table>
    `;
  }

  function loadPages(siteId, container) {
    api('page.list', { siteId }).then(res => {
      if (!res || res.ok !== true) {
        BX.UI.Notification.Center.notify({ content: 'Не удалось загрузить страницы (возможно нет прав)' });
        return;
      }
      renderPages(container, res.pages);
    }).catch(() => BX.UI.Notification.Center.notify({ content: 'Ошибка page.list' }));
  }

  function openCreatePageDialog(siteId, pagesContainer) {
    const formHtml = `
      <div style="display:flex; flex-direction:column; gap:10px;">
        <div>
          <div style="font-size:12px;color:#6a737f;margin-bottom:4px;">Заголовок страницы</div>
          <input type="text" id="pg_title"
            style="width:100%; padding:8px; border:1px solid #d0d7de; border-radius:8px;"
            placeholder="Например: Главная" />
        </div>
        <div>
          <div style="font-size:12px;color:#6a737f;margin-bottom:4px;">Slug (необязательно)</div>
          <input type="text" id="pg_slug"
            style="width:100%; padding:8px; border:1px solid #d0d7de; border-radius:8px;"
            placeholder="home (если пусто — сделаем автоматически)" />
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

        if (!title) {
          BX.UI.Notification.Center.notify({ content: 'Введите заголовок страницы' });
          return;
        }

        api('page.create', { siteId, title, slug }).then(res => {
          if (!res || res.ok !== true) {
            BX.UI.Notification.Center.notify({ content: 'Не удалось создать страницу (возможно нет прав)' });
            return;
          }
          BX.UI.Notification.Center.notify({
            content: `Страница создана: ${BX.util.htmlspecialchars(res.page.title)} (${BX.util.htmlspecialchars(res.page.slug)})`
          });
          mb.close();
          loadPages(siteId, pagesContainer);
        }).catch(() => BX.UI.Notification.Center.notify({ content: 'Ошибка page.create' }));
      }
    });
  }

  function openPagesDialog(siteId, siteName) {
    const html = `
      <div>
        <div style="margin-bottom:10px;">
          <button class="ui-btn ui-btn-primary ui-btn-xs" id="btnCreatePage">Создать страницу</button>
        </div>
        <div id="pagesBox"></div>
      </div>
    `;

    BX.UI.Dialogs.MessageBox.show({
      title: 'Страницы сайта: ' + BX.util.htmlspecialchars(siteName),
      message: html,
      buttons: BX.UI.Dialogs.MessageBoxButtons.CLOSE
    });

    setTimeout(function () {
      const container = document.getElementById('pagesBox');
      if (!container) return;

      loadPages(siteId, container);

      const btn = document.getElementById('btnCreatePage');
      if (!btn) return;

      btn.addEventListener('click', function () {
        openCreatePageDialog(siteId, container);
      });
    }, 0);
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
      if (!res || res.ok !== true) {
        BX.UI.Notification.Center.notify({ content: 'Нет прав на просмотр/изменение доступов (нужен OWNER)' });
        return;
      }
      renderAccess(container, res.access, siteId);
    }).catch(() => BX.UI.Notification.Center.notify({ content: 'Ошибка access.list' }));
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
            <input type="number" id="acc_user_id"
              style="width:100%; padding:8px; border:1px solid #d0d7de; border-radius:8px;"
              placeholder="Например: 15" />
          </div>
          <div style="min-width:160px;">
            <div style="font-size:12px;color:#6a737f;margin-bottom:4px;">Role</div>
            <select id="acc_role" style="width:100%; padding:8px; border:1px solid #d0d7de; border-radius:8px;">
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

        if (!userId) {
          BX.UI.Notification.Center.notify({ content: 'Введите UserID' });
          return;
        }

        api('access.set', { siteId, userId, role }).then(res => {
          if (!res || res.ok !== true) {
            BX.UI.Notification.Center.notify({ content: 'Не удалось назначить доступ (нужен OWNER)' });
            return;
          }
          BX.UI.Notification.Center.notify({ content: 'Доступ назначен' });
          loadAccess(siteId, box);
          loadSites();
        }).catch(() => BX.UI.Notification.Center.notify({ content: 'Ошибка access.set' }));
      });
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
            if (!res || res.ok !== true) {
              BX.UI.Notification.Center.notify({ content: 'Не удалось удалить сайт (нужен OWNER)' });
              return;
            }
            BX.UI.Notification.Center.notify({ content: 'Сайт удалён' });
            mb.close();
            loadSites();
          }).catch(() => BX.UI.Notification.Center.notify({ content: 'Ошибка site.delete' }));
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

    const delPageBtn = e.target.closest('[data-delete-page-id]');
    if (delPageBtn) {
      const id = parseInt(delPageBtn.getAttribute('data-delete-page-id'), 10);
      const siteId = parseInt(delPageBtn.getAttribute('data-delete-page-site-id'), 10);
      if (!id || !siteId) return;

      BX.UI.Dialogs.MessageBox.show({
        title: 'Удалить страницу?',
        message: 'Нужны права EDITOR/ADMIN/OWNER. Продолжить?',
        buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
        onOk: function (mb) {
          api('page.delete', { id }).then(res => {
            if (!res || res.ok !== true) {
              BX.UI.Notification.Center.notify({ content: 'Не удалось удалить страницу (возможно нет прав)' });
              return;
            }
            BX.UI.Notification.Center.notify({ content: 'Страница удалена' });
            mb.close();

            const pagesBox = document.getElementById('pagesBox');
            if (pagesBox) loadPages(siteId, pagesBox);
          }).catch(() => BX.UI.Notification.Center.notify({ content: 'Ошибка page.delete' }));
        }
      });
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
            if (!res || res.ok !== true) {
              BX.UI.Notification.Center.notify({ content: 'Не удалось удалить правило (нужен OWNER)' });
              return;
            }
            BX.UI.Notification.Center.notify({ content: 'Удалено' });
            mb.close();

            const accessBox = document.getElementById('accessBox');
            if (accessBox) loadAccess(siteId, accessBox);
            loadSites();
          }).catch(() => BX.UI.Notification.Center.notify({ content: 'Ошибка access.delete' }));
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
