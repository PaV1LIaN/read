Отлично. Следующий логичный шаг — сделать перетаскивание компонентов мышкой между секциями и колонками.

Сначала делаем простую, но полезную версию:

зажал блок → перетащил в нужную колонку → отпустил → блок перенёсся

Пока без изменения порядка внутри колонки. Сначала перенос между колонками/секциями, потом отдельно сделаем сортировку внутри колонки.

Что сейчас правим

Нужно изменить только 2 файла:

/local/sitebuilder/assets/admin/editor.js
/local/sitebuilder/assets/admin/editor.css


---

1. В editor.js делаем блоки перетаскиваемыми

Найди в renderBlocks() все места, где создаётся блок:

'<div class="sb-editor-block' + active + '" data-block-id="' + Number(block.id || 0) + '">'

Таких мест обычно 2.

Замени каждое на:

'<div class="sb-editor-block' + active + '" draggable="true" data-block-id="' + Number(block.id || 0) + '">'

То есть мы просто добавляем:

draggable="true"


---

2. В editor.js добавь переменную в state

Вверху в state найди:

currentColumn: 1,

Сразу после добавь:

draggedBlockId: 0,

Должно стать так:

currentSectionId: 0,
currentColumn: 1,
draggedBlockId: 0,
accessItems: [],


---

3. В editor.js добавь обработчики drag-and-drop

Найди место, где у тебя уже есть:

blocksList.addEventListener('click', function (e) {

После всего этого обработчика, то есть после его закрытия:

});

вставь:

blocksList.addEventListener('dragstart', function (e) {
    var blockNode = e.target.closest('.sb-editor-block[data-block-id]');
    if (!blockNode) {
        return;
    }

    var blockId = Number(blockNode.getAttribute('data-block-id') || 0);

    if (blockId <= 0) {
        return;
    }

    state.draggedBlockId = blockId;
    blockNode.classList.add('is-dragging');

    if (e.dataTransfer) {
        e.dataTransfer.effectAllowed = 'move';
        e.dataTransfer.setData('text/plain', String(blockId));
    }
});

blocksList.addEventListener('dragend', function (e) {
    var blockNode = e.target.closest('.sb-editor-block[data-block-id]');
    if (blockNode) {
        blockNode.classList.remove('is-dragging');
    }

    state.draggedBlockId = 0;

    blocksList.querySelectorAll('.sb-editor-section-preview__column.is-drag-over').forEach(function (columnNode) {
        columnNode.classList.remove('is-drag-over');
    });
});

blocksList.addEventListener('dragover', function (e) {
    var columnNode = e.target.closest('.sb-editor-section-preview__column[data-section-id][data-column]');
    if (!columnNode) {
        return;
    }

    e.preventDefault();

    if (e.dataTransfer) {
        e.dataTransfer.dropEffect = 'move';
    }

    blocksList.querySelectorAll('.sb-editor-section-preview__column.is-drag-over').forEach(function (node) {
        if (node !== columnNode) {
            node.classList.remove('is-drag-over');
        }
    });

    columnNode.classList.add('is-drag-over');
});

blocksList.addEventListener('dragleave', function (e) {
    var columnNode = e.target.closest('.sb-editor-section-preview__column[data-section-id][data-column]');
    if (!columnNode) {
        return;
    }

    var related = e.relatedTarget;

    if (related && columnNode.contains(related)) {
        return;
    }

    columnNode.classList.remove('is-drag-over');
});

blocksList.addEventListener('drop', async function (e) {
    var columnNode = e.target.closest('.sb-editor-section-preview__column[data-section-id][data-column]');
    if (!columnNode) {
        return;
    }

    e.preventDefault();

    columnNode.classList.remove('is-drag-over');

    var blockId = Number(state.draggedBlockId || 0);

    if (!blockId && e.dataTransfer) {
        blockId = Number(e.dataTransfer.getData('text/plain') || 0);
    }

    var sectionId = Number(columnNode.getAttribute('data-section-id') || 0);
    var column = Number(columnNode.getAttribute('data-column') || 1);

    if (blockId <= 0 || sectionId <= 0) {
        return;
    }

    try {
        await assignBlockToSection(blockId, sectionId, column);

        state.currentBlockId = blockId;
        state.currentSectionId = sectionId;
        state.currentColumn = column;

        await loadBlocks();

        setPageSectionsMessage('Блок перенесён в секцию #' + sectionId + ', колонку ' + column, 'success');
    } catch (err) {
        console.error(err);
        setPageSectionsMessage('Не удалось перенести блок', 'error');
    }
});


---

4. В editor.css добавь стили

Файл:

/local/sitebuilder/assets/admin/editor.css

В самый конец добавь:

/* =========================================================
   Drag and drop blocks
   ========================================================= */

.sb-editor-block[draggable="true"] {
    cursor: grab;
}

.sb-editor-block[draggable="true"]:active {
    cursor: grabbing;
}

.sb-editor-block.is-dragging {
    opacity: 0.45;
    transform: scale(0.98);
}

.sb-editor-section-preview__column.is-drag-over {
    outline: 2px dashed #2563eb;
    outline-offset: -6px;
    background: rgba(37, 99, 235, 0.06);
}

.sb-editor-section-preview__column.is-drag-over .sb-editor-section-preview__empty {
    color: #2563eb;
    border-color: rgba(37, 99, 235, 0.35);
    background: rgba(37, 99, 235, 0.08);
}


---

5. Обнови версию JS/CSS

В editor.php найди подключение:

/assets/admin/editor.js?v=...

Поставь новую версию, например:

<script src="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor.js?v=2"></script>

Если editor.css подключён без версии, лучше тоже сделать:

<link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor.css?v=2">


---

Что должно заработать

Теперь в редакторе:

1. зажимаешь блок мышкой;


2. тащишь в другую колонку;


3. колонка подсвечивается;


4. отпускаешь;


5. блок переносится в эту секцию/колонку;


6. после обновления страницы блок остаётся там же.



После этого следующим шагом сделаем изменение порядка блоков внутри одной колонки перетаскиванием, чтобы можно было ставить блок выше/ниже прямо мышкой.