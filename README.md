Идём дальше.

Следующий файл — editor.api.js.

Создай:

/local/sitebuilder/assets/js/editor.api.js

и вставь туда это:

window.SBEditor = window.SBEditor || {};

window.SBEditor.notify = function (message) {
  BX.UI.Notification.Center.notify({
    content: BX.util.htmlspecialchars(String(message || ''))
  });
};

window.SBEditor.api = function (action, data = {}) {
  return new Promise((resolve, reject) => {
    BX.ajax.runComponentAction?.(); // безопасно не используем, просто чтобы ничего не ломать

    BX.ajax({
      url: '/local/sitebuilder/api.php',
      method: 'POST',
      dataType: 'json',
      data: Object.assign({}, data, {
        action,
        sessid: BX.bitrix_sessid()
      }),
      onsuccess: function (res) {
        resolve(res);
      },
      onfailure: function (err) {
        reject(err);
      }
    });
  });
};

window.SBEditor.fileDownloadUrl = function (siteId, fileId) {
  return '/local/sitebuilder/download.php?siteId=' + encodeURIComponent(siteId) + '&fileId=' + encodeURIComponent(fileId);
};

window.SBEditor.getFilesForSite = async function () {
  const st = window.SBEditor.getState();
  const res = await window.SBEditor.api('file.list', { siteId: st.siteId });
  if (!res || res.ok !== true) {
    throw new Error('FILE_LIST_FAILED');
  }
  return Array.isArray(res.files) ? res.files : [];
};

window.SBEditor.btnClass = function (kind) {
  switch (kind) {
    case 'primary':
      return 'ui-btn-primary';
    case 'danger':
      return 'ui-btn-danger';
    case 'success':
      return 'ui-btn-success';
    case 'warning':
      return 'ui-btn-warning';
    default:
      return 'ui-btn-light';
  }
};

window.SBEditor.headingTag = function (level) {
  return ['h1', 'h2', 'h3'].includes(level) ? level : 'h2';
};

window.SBEditor.headingAlign = function (align) {
  return ['left', 'center', 'right'].includes(align) ? align : 'left';
};

window.SBEditor.colsGridTemplate = function (ratio) {
  if (ratio === '33-67') return '1fr 2fr';
  if (ratio === '67-33') return '2fr 1fr';
  return '1fr 1fr';
};

window.SBEditor.galleryTemplate = function (columns) {
  const cols = parseInt(columns || 3, 10);
  if (cols === 2) return '1fr 1fr';
  if (cols === 4) return '1fr 1fr 1fr 1fr';
  return '1fr 1fr 1fr';
};

window.SBEditor.cardsNormalizeItem = function (item = {}) {
  return {
    title: String(item.title || '').trim(),
    text: String(item.text || ''),
    imageFileId: parseInt(item.imageFileId || 0, 10) || 0,
    buttonText: String(item.buttonText || '').trim(),
    buttonUrl: String(item.buttonUrl || '').trim()
  };
};

Что дальше

После этого следующий правильный шаг — editor.sections.js, потому что:

у тебя section уже есть

логика section хорошо выделяется

её безопаснее всего вынести первой после API


Когда вставишь editor.api.js, напиши, и я дам готовый editor.sections.js.