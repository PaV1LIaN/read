Да, сейчас модалка развалилась из-за того, что форма настроек техническая: поля и чекбоксы идут как попало. Сделаем нормальную структуру:

сверху заголовок и аккуратная кнопка закрытия;

основные поля в 2 колонки: слева название, справа поле;

чекбоксы отдельным красивым блоком “Возможности”;

кнопки “Отмена / Сохранить” внизу справа;

модалка меньше по высоте и с нормальными отступами.


1. Правка script.js

Файл:

/local/sitebuilder/components/disk/script.js

Найди в openSettingsModal() вот этот кусок:

this.fillSettingsForm(
  settingsRes.data.settings || {},
  rootOptionsRes.data || {}
);

this.setSettingsMessage('');

Замени на:

this.fillSettingsForm(
  settingsRes.data.settings || {},
  rootOptionsRes.data || {}
);

this.arrangeSettingsModal();

this.setSettingsMessage('');

Теперь ниже метода openSettingsModal, перед:

DiskComponent.prototype.closeSettingsModal = function () {

вставь новый метод:

DiskComponent.prototype.arrangeSettingsModal = function () {
  var modal = this.root.querySelector('[data-role="settings-modal"]');
  var form = this.root.querySelector('[data-role="settings-form"]');

  if (!modal || !form) {
    return;
  }

  modal.classList.add('sb-disk-settings-modal');

  var shell = modal.firstElementChild;
  if (shell) {
    shell.classList.add('sb-disk-settings-shell');
  }

  if (!form.querySelector('.sb-disk-settings-section-main')) {
    var mainTitle = document.createElement('div');
    mainTitle.className = 'sb-disk-settings-section-main';
    mainTitle.textContent = 'Основные настройки';

    form.insertBefore(mainTitle, form.firstChild);
  }

  var checkboxLabels = Array.prototype.slice.call(
    form.querySelectorAll('label')
  ).filter(function (label) {
    return !!label.querySelector('input[type="checkbox"]');
  });

  if (checkboxLabels.length && !form.querySelector('.sb-disk-settings-checks')) {
    var checksTitle = document.createElement('div');
    checksTitle.className = 'sb-disk-settings-section-title';
    checksTitle.textContent = 'Возможности';

    var checksWrap = document.createElement('div');
    checksWrap.className = 'sb-disk-settings-checks';

    checkboxLabels.forEach(function (label) {
      checksWrap.appendChild(label);
    });

    form.appendChild(checksTitle);
    form.appendChild(checksWrap);
  }

  var actionButtons = Array.prototype.slice.call(
    modal.querySelectorAll('[data-action="save-settings"], [data-action="close-settings"]')
  ).filter(function (button) {
    var text = String(button.textContent || '').trim().toLowerCase();

    return text !== '×' && text !== 'x';
  });

  if (actionButtons.length && !modal.querySelector('.sb-disk-settings-footer')) {
    var footer = document.createElement('div');
    footer.className = 'sb-disk-settings-footer';

    actionButtons.forEach(function (button) {
      footer.appendChild(button);
    });

    if (shell) {
      shell.appendChild(footer);
    } else {
      modal.appendChild(footer);
    }
  }
};


---

2. Правка styles.css

Файл:

/local/sitebuilder/components/disk/styles.css

В самый конец файла добавь этот блок:

/* =========================================================
   SETTINGS MODAL FINAL NORMAL VIEW
   ========================================================= */

.sb-disk-settings-modal {
    position: fixed !important;
    inset: 0 !important;
    z-index: 10000 !important;
    display: flex !important;
    align-items: center !important;
    justify-content: center !important;
    padding: 24px !important;
    background: rgba(15, 23, 42, .50) !important;
    backdrop-filter: blur(7px);
    overflow: auto !important;
}

.sb-disk-settings-modal[hidden] {
    display: none !important;
}

.sb-disk-settings-shell {
    position: relative !important;
    width: min(760px, calc(100vw - 48px)) !important;
    max-height: calc(100vh - 48px) !important;
    margin: 0 !important;
    padding: 26px !important;
    overflow: auto !important;
    border: 1px solid rgba(226, 232, 240, .95) !important;
    border-radius: 24px !important;
    background: #ffffff !important;
    box-shadow: 0 34px 90px rgba(15, 23, 42, .30) !important;
}

/* Заголовок */
.sb-disk-settings-modal h1,
.sb-disk-settings-modal h2,
.sb-disk-settings-modal h3 {
    margin: 0 48px 22px 0 !important;
    color: #111827 !important;
    font-size: 24px !important;
    line-height: 1.2 !important;
    font-weight: 900 !important;
}

/* Кнопка закрытия X */
.sb-disk-settings-modal [data-action="close-settings"] {
    min-height: 36px !important;
    height: 36px !important;
    padding: 0 14px !important;
    border-radius: 12px !important;
    border: 1px solid #dbe3ef !important;
    background: #ffffff !important;
    color: #374151 !important;
    font-size: 13px !important;
    font-weight: 800 !important;
}

.sb-disk-settings-modal [data-action="close-settings"]:not(.sb-disk-settings-footer [data-action="close-settings"]) {
    position: absolute !important;
    top: 20px !important;
    right: 20px !important;
    width: 36px !important;
    min-width: 36px !important;
    padding: 0 !important;
    font-size: 0 !important;
}

.sb-disk-settings-modal [data-action="close-settings"]:not(.sb-disk-settings-footer [data-action="close-settings"])::before {
    content: "×";
    font-size: 20px;
    line-height: 1;
}

/* Форма */
.sb-disk-settings-modal [data-role="settings-form"] {
    display: grid !important;
    grid-template-columns: 210px minmax(0, 1fr) !important;
    gap: 12px 16px !important;
    align-items: center !important;
    margin: 0 !important;
    padding: 0 !important;
}

/* Разделы */
.sb-disk-settings-section-main,
.sb-disk-settings-section-title {
    grid-column: 1 / -1 !important;
    margin-top: 6px !important;
    padding-top: 6px !important;
    color: #111827 !important;
    font-size: 15px !important;
    font-weight: 900 !important;
}

.sb-disk-settings-section-title {
    margin-top: 18px !important;
    padding-top: 18px !important;
    border-top: 1px solid #eef2f7 !important;
}

/* Обычные label */
.sb-disk-settings-modal [data-role="settings-form"] > label:not(:has(input[type="checkbox"])) {
    margin: 0 !important;
    color: #374151 !important;
    font-size: 13px !important;
    font-weight: 800 !important;
    line-height: 1.3 !important;
}

/* Поля */
.sb-disk-settings-modal [data-role="settings-form"] input[type="text"],
.sb-disk-settings-modal [data-role="settings-form"] input[type="number"],
.sb-disk-settings-modal [data-role="settings-form"] select,
.sb-disk-settings-modal [data-role="settings-form"] textarea {
    width: 100% !important;
    min-width: 0 !important;
    max-width: 100% !important;
    height: 40px !important;
    padding: 0 13px !important;
    border: 1px solid #dbe3ef !important;
    border-radius: 13px !important;
    background: #ffffff !important;
    color: #111827 !important;
    font-size: 13px !important;
    outline: none !important;
    box-shadow: none !important;
}

.sb-disk-settings-modal [data-role="settings-form"] textarea {
    height: auto !important;
    min-height: 84px !important;
    padding-top: 10px !important;
    padding-bottom: 10px !important;
}

.sb-disk-settings-modal [data-role="settings-form"] input:focus,
.sb-disk-settings-modal [data-role="settings-form"] select:focus,
.sb-disk-settings-modal [data-role="settings-form"] textarea:focus {
    border-color: var(--disk-accent, #2563eb) !important;
    box-shadow: 0 0 0 3px rgba(37, 99, 235, .12) !important;
}

/* Подсказки */
.sb-disk-settings-modal [data-role="settings-form"] small,
.sb-disk-settings-modal [data-role="settings-form"] .hint,
.sb-disk-settings-modal [data-role="settings-form"] .help {
    grid-column: 2 / 3 !important;
    margin: -6px 0 4px !important;
    color: #6b7280 !important;
    font-size: 12px !important;
    line-height: 1.35 !important;
}

/* Чекбоксы отдельной красивой сеткой */
.sb-disk-settings-checks {
    grid-column: 1 / -1 !important;
    display: grid !important;
    grid-template-columns: repeat(2, minmax(0, 1fr)) !important;
    gap: 8px !important;
    margin-top: 0 !important;
}

.sb-disk-settings-checks label {
    display: flex !important;
    align-items: center !important;
    gap: 8px !important;
    min-height: 38px !important;
    margin: 0 !important;
    padding: 9px 10px !important;
    border: 1px solid #e5e7eb !important;
    border-radius: 13px !important;
    background: #f8fafc !important;
    color: #374151 !important;
    font-size: 12px !important;
    font-weight: 700 !important;
    line-height: 1.3 !important;
}

.sb-disk-settings-checks label:hover {
    border-color: #c7d2fe !important;
    background: #f8fbff !important;
}

.sb-disk-settings-checks input[type="checkbox"] {
    width: 16px !important;
    height: 16px !important;
    margin: 0 !important;
    flex: 0 0 16px !important;
    accent-color: var(--disk-accent, #2563eb);
}

/* Низ модалки */
.sb-disk-settings-footer {
    display: flex !important;
    align-items: center !important;
    justify-content: flex-end !important;
    gap: 10px !important;
    margin-top: 22px !important;
    padding-top: 18px !important;
    border-top: 1px solid #eef2f7 !important;
}

.sb-disk-settings-footer button {
    min-height: 40px !important;
    padding: 0 16px !important;
    border-radius: 12px !important;
    font-size: 13px !important;
    font-weight: 800 !important;
}

.sb-disk-settings-footer [data-action="save-settings"] {
    border-color: var(--disk-accent, #2563eb) !important;
    background: var(--disk-accent, #2563eb) !important;
    color: #fff !important;
}

.sb-disk-settings-footer [data-action="save-settings"]:hover {
    background: var(--disk-accent-hover, #3f5de9) !important;
    border-color: var(--disk-accent-hover, #3f5de9) !important;
    color: #fff !important;
}

/* Сообщение */
.sb-disk-settings-modal [data-role="settings-message"] {
    margin-top: 12px !important;
    color: #6b7280 !important;
    font-size: 13px !important;
}

/* Адаптив */
@media (max-width: 760px) {
    .sb-disk-settings-modal {
        align-items: flex-start !important;
        padding: 12px !important;
    }

    .sb-disk-settings-shell {
        width: 100% !important;
        max-height: none !important;
        padding: 18px !important;
        border-radius: 20px !important;
    }

    .sb-disk-settings-modal [data-role="settings-form"] {
        grid-template-columns: 1fr !important;
        gap: 8px !important;
    }

    .sb-disk-settings-modal [data-role="settings-form"] small,
    .sb-disk-settings-modal [data-role="settings-form"] .hint,
    .sb-disk-settings-modal [data-role="settings-form"] .help {
        grid-column: auto !important;
        margin: -2px 0 4px !important;
    }

    .sb-disk-settings-checks {
        grid-template-columns: 1fr !important;
    }

    .sb-disk-settings-footer {
        align-items: stretch !important;
        flex-direction: column-reverse !important;
    }

    .sb-disk-settings-footer button {
        width: 100% !important;
    }
}

3. Обнови версии подключения

В public_page.php поставь новые версии:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/components/disk/styles.css?v=10">

<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js?v=8"></script>

После этого сделай Ctrl + F5.

Настройки должны стать похожи на нормальную админскую форму: поля слева/справа, чекбоксы отдельной сеткой, кнопки снизу, без этой каши посередине.