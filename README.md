Да, следующий шаг — добавляем иконки файлов/папок и красивую пустую заглушку через JS, не ломая текущую логику диска.

Сейчас просто добавь код в самый конец файла:

/local/sitebuilder/components/disk/script.js

/* =========================================================
   SITEBUILDER DISK UI ENHANCER
   Красивые иконки, пустое состояние, кнопки
   Не ломает основную логику диска
   ========================================================= */

(function () {
    'use strict';

    function text(node) {
        return String(node && node.textContent ? node.textContent : '').trim();
    }

    function lower(value) {
        return String(value || '').toLowerCase();
    }

    function getFileExtension(name) {
        name = String(name || '').trim();

        var cleanName = name.split('?')[0].split('#')[0];
        var parts = cleanName.split('.');

        if (parts.length < 2) {
            return '';
        }

        return lower(parts.pop());
    }

    function iconByExt(ext, isFolder) {
        if (isFolder) {
            return '📁';
        }

        if (ext === 'pdf') {
            return 'PDF';
        }

        if (['doc', 'docx', 'rtf'].indexOf(ext) !== -1) {
            return 'DOC';
        }

        if (['xls', 'xlsx', 'csv'].indexOf(ext) !== -1) {
            return 'XLS';
        }

        if (['ppt', 'pptx'].indexOf(ext) !== -1) {
            return 'PPT';
        }

        if (['jpg', 'jpeg', 'png', 'gif', 'webp', 'svg'].indexOf(ext) !== -1) {
            return 'IMG';
        }

        if (['zip', 'rar', '7z'].indexOf(ext) !== -1) {
            return 'ZIP';
        }

        if (['txt', 'log'].indexOf(ext) !== -1) {
            return 'TXT';
        }

        return 'FILE';
    }

    function iconClassByExt(ext, isFolder) {
        if (isFolder) {
            return 'sb-disk-icon sb-disk-icon-folder';
        }

        if (ext === 'pdf') {
            return 'sb-disk-icon sb-disk-icon-pdf';
        }

        if (['doc', 'docx', 'rtf'].indexOf(ext) !== -1) {
            return 'sb-disk-icon sb-disk-icon-doc';
        }

        if (['xls', 'xlsx', 'csv'].indexOf(ext) !== -1) {
            return 'sb-disk-icon sb-disk-icon-xls';
        }

        if (['ppt', 'pptx'].indexOf(ext) !== -1) {
            return 'sb-disk-icon sb-disk-icon-ppt';
        }

        if (['jpg', 'jpeg', 'png', 'gif', 'webp', 'svg'].indexOf(ext) !== -1) {
            return 'sb-disk-icon sb-disk-icon-img';
        }

        if (['zip', 'rar', '7z'].indexOf(ext) !== -1) {
            return 'sb-disk-icon sb-disk-icon-zip';
        }

        return 'sb-disk-icon sb-disk-icon-file';
    }

    function isFolderRow(row) {
        var rowText = lower(text(row));

        if (row.getAttribute('data-type') === 'folder') {
            return true;
        }

        if (row.classList.contains('is-folder') || row.classList.contains('sb-disk-row-folder')) {
            return true;
        }

        return rowText.indexOf('папка') !== -1 || rowText.indexOf('folder') !== -1;
    }

    function findNameCell(row) {
        var cells = Array.prototype.slice.call(row.querySelectorAll('td'));

        if (!cells.length) {
            return null;
        }

        for (var i = 0; i < cells.length; i++) {
            var cell = cells[i];

            if (cell.querySelector('input[type="checkbox"]') && text(cell).length < 3) {
                continue;
            }

            if (text(cell) !== '') {
                return cell;
            }
        }

        return cells[0];
    }

    function enhanceTableRows(disk) {
        var rows = disk.querySelectorAll('tbody tr');

        rows.forEach(function (row) {
            if (row.classList.contains('sb-disk-row-enhanced')) {
                return;
            }

            var nameCell = findNameCell(row);

            if (!nameCell) {
                return;
            }

            var name = text(nameCell);

            if (!name) {
                return;
            }

            var isFolder = isFolderRow(row);
            var ext = getFileExtension(name);

            row.classList.add('sb-disk-row-enhanced');

            if (isFolder) {
                row.classList.add('sb-disk-row-folder');
                row.setAttribute('data-type', 'folder');
            }

            if (ext) {
                row.setAttribute('data-ext', ext);
            }

            var wrapper = document.createElement('span');
            wrapper.className = 'sb-disk-name-wrap';

            var icon = document.createElement('span');
            icon.className = iconClassByExt(ext, isFolder);
            icon.textContent = iconByExt(ext, isFolder);

            var label = document.createElement('span');
            label.className = 'sb-disk-name-label';

            while (nameCell.firstChild) {
                label.appendChild(nameCell.firstChild);
            }

            wrapper.appendChild(icon);
            wrapper.appendChild(label);
            nameCell.appendChild(wrapper);
        });
    }

    function enhanceButtons(disk) {
        var buttons = disk.querySelectorAll('button, .sb-btn');

        buttons.forEach(function (button) {
            var value = lower(text(button));

            if (value.indexOf('загруз') !== -1) {
                button.classList.add('sb-disk-upload-btn', 'is-primary');
                button.setAttribute('data-disk-ui', 'upload');
            }

            if (value.indexOf('новая пап') !== -1 || value.indexOf('создать пап') !== -1) {
                button.classList.add('sb-disk-folder-btn');
                button.setAttribute('data-disk-ui', 'folder');
            }

            if (value.indexOf('таблица') !== -1 || value.indexOf('плитка') !== -1) {
                button.classList.add('sb-disk-view-btn');
            }
        });
    }

    function enhanceInputs(disk) {
        var searchInputs = disk.querySelectorAll('input[type="text"], input[type="search"]');

        searchInputs.forEach(function (input) {
            var placeholder = input.getAttribute('placeholder') || '';

            if (!placeholder) {
                input.setAttribute('placeholder', 'Поиск файлов и папок');
            }

            input.classList.add('sb-disk-search-input');
        });

        var selects = disk.querySelectorAll('select');

        selects.forEach(function (select) {
            select.classList.add('sb-disk-select');
        });
    }

    function enhanceEmptyState(disk) {
        var loading = disk.querySelector('.sb-public-disk-loading');

        if (!loading) {
            return;
        }

        var value = lower(text(loading));

        if (value.indexOf('загрузка') !== -1) {
            return;
        }

        if (
            value.indexOf('нет файлов') === -1 &&
            value.indexOf('пуст') === -1 &&
            value.indexOf('здесь пока нет') === -1
        ) {
            return;
        }

        if (loading.classList.contains('sb-disk-empty-enhanced')) {
            return;
        }

        loading.classList.add('sb-disk-empty-enhanced');
        loading.innerHTML = ''
            + '<div class="sb-disk-empty-icon">📁</div>'
            + '<strong>Пока здесь пусто</strong>'
            + '<span>Загрузите первый файл или создайте новую папку.</span>'
            + '<div class="sb-disk-empty-actions">'
            + '    <button type="button" class="sb-disk-empty-upload">Загрузить файл</button>'
            + '</div>';
    }

    function bindEmptyUpload(disk) {
        if (disk.getAttribute('data-empty-upload-bound') === '1') {
            return;
        }

        disk.setAttribute('data-empty-upload-bound', '1');

        disk.addEventListener('click', function (e) {
            var btn = e.target.closest('.sb-disk-empty-upload');

            if (!btn) {
                return;
            }

            var fileInput = disk.querySelector('input[type="file"]');

            if (fileInput) {
                fileInput.click();
                return;
            }

            var uploadBtn = disk.querySelector(
                '.sb-disk-upload-btn, button[data-disk-ui="upload"], button[data-action="upload"], button[data-disk-action="upload"]'
            );

            if (uploadBtn) {
                uploadBtn.click();
            }
        });
    }

    function enhanceDisk(disk) {
        if (!disk) {
            return;
        }

        disk.classList.add('sb-disk-enhanced');

        enhanceButtons(disk);
        enhanceInputs(disk);
        enhanceTableRows(disk);
        enhanceEmptyState(disk);
        bindEmptyUpload(disk);
    }

    function enhanceAllDisks() {
        document.querySelectorAll('.sb-disk').forEach(enhanceDisk);
    }

    function observeDisks() {
        var timer = null;

        var observer = new MutationObserver(function () {
            clearTimeout(timer);

            timer = setTimeout(function () {
                enhanceAllDisks();
            }, 80);
        });

        observer.observe(document.body, {
            childList: true,
            subtree: true
        });
    }

    document.addEventListener('DOMContentLoaded', function () {
        enhanceAllDisks();
        observeDisks();
    });

    if (document.readyState === 'interactive' || document.readyState === 'complete') {
        enhanceAllDisks();
        observeDisks();
    }
})();

Теперь в конец файла:

/local/sitebuilder/components/disk/styles.css

добавь ещё этот блок:

/* =========================================================
   DISK UI ENHANCER ADDITIONS
   ========================================================= */

.sb-disk-name-wrap {
    display: inline-flex;
    align-items: center;
    gap: 10px;
    min-width: 0;
    max-width: 100%;
}

.sb-disk-name-label {
    min-width: 0;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}

.sb-disk-icon {
    width: 34px;
    height: 34px;
    min-width: 34px;
    max-width: 34px;
    min-height: 34px;
    max-height: 34px;
    border-radius: 12px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    background: #eef2ff;
    color: var(--disk-accent, #2563eb);
    font-size: 10px;
    font-weight: 900;
    line-height: 1;
    flex: 0 0 34px;
}

.sb-disk-icon-folder {
    background: #fef3c7;
    color: #92400e;
    font-size: 18px;
}

.sb-disk-icon-pdf {
    background: #fee2e2;
    color: #991b1b;
}

.sb-disk-icon-doc {
    background: #dbeafe;
    color: #1d4ed8;
}

.sb-disk-icon-xls {
    background: #dcfce7;
    color: #166534;
}

.sb-disk-icon-ppt {
    background: #ffedd5;
    color: #c2410c;
}

.sb-disk-icon-img {
    background: #fce7f3;
    color: #be185d;
}

.sb-disk-icon-zip {
    background: #ede9fe;
    color: #6d28d9;
}

.sb-disk-icon-file {
    background: #f1f5f9;
    color: #475569;
}

.sb-disk-row-enhanced td {
    vertical-align: middle;
}

.sb-disk-empty-enhanced {
    min-height: 190px !important;
    padding: 34px !important;
}

.sb-disk-empty-enhanced::before {
    display: none !important;
}

.sb-disk-empty-icon {
    width: 64px;
    height: 64px;
    border-radius: 22px;
    background: #eef2ff;
    color: var(--disk-accent, #2563eb);
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 32px;
    margin-bottom: 6px;
}

.sb-disk-empty-enhanced strong {
    font-size: 17px;
    line-height: 1.25;
    color: #111827;
}

.sb-disk-empty-enhanced span {
    max-width: 360px;
    color: #6b7280;
    font-size: 13px;
    line-height: 1.45;
}

.sb-disk-empty-actions {
    margin-top: 8px;
    display: flex;
    justify-content: center;
}

.sb-disk-empty-upload {
    border-color: var(--disk-accent, #2563eb) !important;
    background: var(--disk-accent, #2563eb) !important;
    color: #fff !important;
}

И в public_page.php обнови версию подключения:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/components/disk/styles.css?v=3">

А подключение JS лучше так:

<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js?v=2"></script>

После этого сделай Ctrl + F5.

Что должно измениться:

1. У файлов появятся иконки PDF/DOC/XLS/IMG/ZIP/FILE.
2. У папок будет отдельная иконка папки.
3. Пустой диск будет выглядеть как нормальная карточка с кнопкой “Загрузить файл”.
4. Кнопка “Загрузить” станет основной синей.
5. Поиск и селекты станут аккуратнее.