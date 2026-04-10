Понял. Скорее всего, проблема в том, что после редактирования блок сохраняется, но редактор не перерисовывает форму текущего блока, либо не находится блок по id/zone после обновления state.layout.

Дам тебе сразу рабочую правку.

Что заменить в layout.php

Найди функцию:

function saveLayoutBlock() {

и замени её целиком на эту:

function saveLayoutBlock() {
    if (!state.currentBlockId) {
        return;
    }

    var contentText = document.getElementById('editLayoutBlockContent').value || '{}';
    var propsText = document.getElementById('editLayoutBlockProps').value || '{}';

    try {
        JSON.parse(contentText);
    } catch (e) {
        alert('Content должен быть валидным JSON');
        return;
    }

    try {
        JSON.parse(propsText);
    } catch (e) {
        alert('Props должен быть валидным JSON');
        return;
    }

    var currentId = Number(state.currentBlockId || 0);
    var currentZone = String(state.currentZone || '');

    api('layout.block.update', {
        siteId: SITE_ID,
        id: currentId,
        content: contentText,
        props: propsText
    }, function (res) {
        if (!res || res.ok !== true) {
            alert('Не удалось сохранить layout block');
            return;
        }

        state.layout = res.layout || state.layout;
        renderZones();

        var found = findLayoutBlock(currentId);
        if (found) {
            state.currentBlockId = currentId;
            state.currentZone = found.zone;
            document.getElementById('editLayoutBlockZone').value = found.zone;
            document.getElementById('editLayoutBlockType').value = found.block.type || '';
            document.getElementById('editLayoutBlockContent').value = JSON.stringify(found.block.content || {}, null, 2);
            document.getElementById('editLayoutBlockProps').value = JSON.stringify(found.block.props || {}, null, 2);
            document.getElementById('layoutBlockEditorEmpty').classList.add('hidden');
            document.getElementById('layoutBlockEditorForm').classList.remove('hidden');
        } else {
            state.currentBlockId = 0;
            state.currentZone = currentZone;
            clearLayoutBlockEditor();
        }
    });
}


---

Ещё одна важная правка

Найди функцию:

function editLayoutBlock(zone, blockId) {

и замени её целиком на эту:

function editLayoutBlock(zone, blockId) {
    var found = findLayoutBlock(blockId);
    if (!found) {
        alert('Блок не найден');
        return;
    }

    state.currentBlockId = Number(blockId || 0);
    state.currentZone = found.zone || zone;

    document.getElementById('layoutBlockEditorEmpty').classList.add('hidden');
    document.getElementById('layoutBlockEditorForm').classList.remove('hidden');
    document.getElementById('editLayoutBlockZone').value = state.currentZone;
    document.getElementById('editLayoutBlockType').value = found.block.type || '';
    document.getElementById('editLayoutBlockContent').value = JSON.stringify(found.block.content || {}, null, 2);
    document.getElementById('editLayoutBlockProps').value = JSON.stringify(found.block.props || {}, null, 2);
}


---

И ещё одна мелкая правка

В функции saveLayoutSettings() всё ок.
Но в renderZones() после перерисовки карточек редактор может визуально “остаться на старом блоке”. Чтобы это не ломало UX, найди функцию:

function renderZones() {

и замени на:

function renderZones() {
    if (!state.layout || !state.layout.zones) {
        zonesContainer.innerHTML = '<div class="editor-empty">Нет layout-зон</div>';
        return;
    }

    var html = '';
    for (var i = 0; i < ZONES.length; i++) {
        html += renderZoneCard(ZONES[i], state.layout.zones[ZONES[i]] || []);
    }
    zonesContainer.innerHTML = html;

    if (state.currentBlockId) {
        var found = findLayoutBlock(state.currentBlockId);
        if (!found) {
            clearLayoutBlockEditor();
        }
    }
}


---

Что проверить после правки

1. открыть layout.php


2. добавить блок в header


3. нажать Редактировать


4. поменять content


5. нажать Сохранить



Ожидаемо:

ошибка не появляется

блок сохраняется

редактор остаётся открыт на том же блоке

список блоков обновляется



---

Если под “не работает редактировать” ты имеешь в виду, что кнопка вообще ничего не делает

Тогда проблема, скорее всего, в делегировании клика.
В таком случае замени этот кусок:

var editBtn = e.target.closest('.js-edit-layout-block');
if (editBtn) {
    editLayoutBlock(
        editBtn.getAttribute('data-zone') || '',
        parseInt(editBtn.getAttribute('data-id'), 10) || 0
    );
    return;
}

на такой же, но с отладкой:

var editBtn = e.target.closest('.js-edit-layout-block');
if (editBtn) {
    print({
        action: 'edit-layout-block-click',
        zone: editBtn.getAttribute('data-zone') || '',
        id: parseInt(editBtn.getAttribute('data-id'), 10) || 0
    });

    editLayoutBlock(
        editBtn.getAttribute('data-zone') || '',
        parseInt(editBtn.getAttribute('data-id'), 10) || 0
    );
    return;
}

Если после клика в debug-блоке ничего не появляется, значит проблема именно в обработчике клика.

Если хочешь, следующим сообщением я могу дать тебе уже исправленный полный layout.php целиком, чтобы не править кусками.