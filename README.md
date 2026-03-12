Отлично, делаем drag-and-drop сортировку блоков.

Самый надёжный вариант для твоего проекта:

в editor.php добавить перетаскивание карточек блоков,

в api.php добавить новый action, который принимает массив id в новом порядке,

на сервере пересчитать sort.


Так будет лучше, чем пытаться эмулировать ↑ ↓.

Что добавим

В api.php

Новый action:

if ($action === 'block.reorder') {

Он будет принимать:

pageId

order — JSON-массив id блоков в новом порядке


В editor.php

у контейнера блоков включим drag-and-drop

после отпускания блока отправим новый порядок в api.php



---

1. Добавь в api.php новый action block.reorder

Вставь перед финальным:

http_response_code(400);
echo json_encode(['ok' => false, 'error' => 'UNKNOWN_ACTION', 'action' => $action], JSON_UNESCAPED_UNICODE);

вот этот код:

if ($action === 'block.reorder') {
    $pageId = (int)($_POST['pageId'] ?? 0);
    $orderJson = (string)($_POST['order'] ?? '[]');

    if ($pageId <= 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'PAGE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $page = sb_find_page($pageId);
    if (!$page) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $order = json_decode($orderJson, true);
    if (!is_array($order) || !$order) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'BAD_ORDER'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $orderIds = array_values(array_map('intval', $order));
    $orderMap = [];
    foreach ($orderIds as $pos => $id) {
        if ($id > 0) $orderMap[$id] = $pos;
    }

    $blocks = sb_read_blocks();

    $pageBlocks = array_values(array_filter($blocks, fn($b) => (int)($b['pageId'] ?? 0) === $pageId));
    $pageBlockIds = array_map(fn($b) => (int)($b['id'] ?? 0), $pageBlocks);
    sort($pageBlockIds);

    $submittedIds = array_keys($orderMap);
    sort($submittedIds);

    if ($pageBlockIds !== $submittedIds) {
        http_response_code(422);
        echo json_encode([
            'ok'=>false,
            'error'=>'ORDER_MISMATCH',
            'expected'=>$pageBlockIds,
            'got'=>$submittedIds,
        ], JSON_UNESCAPED_UNICODE);
        exit;
    }

    foreach ($blocks as &$b) {
        $bid = (int)($b['id'] ?? 0);
        if ((int)($b['pageId'] ?? 0) !== $pageId) continue;
        if (!isset($orderMap[$bid])) continue;

        $b['sort'] = ($orderMap[$bid] + 1) * 10;
        $b['updatedAt'] = date('c');
        $b['updatedBy'] = (int)$USER->GetID();
    }
    unset($b);

    sb_write_blocks($blocks);

    echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE);
    exit;
}


---

2. В editor.php добавь стили для перетаскивания

В <style> добавь:

.dragHandle {
  cursor: grab;
  user-select: none;
}

.dragHandle:active {
  cursor: grabbing;
}

.block.dragging {
  opacity: .55;
  border-color:#93c5fd;
}

.dropMarker {
  height: 10px;
  margin-top: 10px;
  border-radius: 999px;
  background: #bfdbfe;
}


---

3. В buildBlockShell() добавь handle

Найди в buildBlockShell вот этот кусок:

<div class="blockTitleRow">
  <b>#${id}</b>
  <span class="blockTypeBadge">${BX.util.htmlspecialchars(type)}</span>
</div>

и замени на:

<div class="blockTitleRow">
  <span class="dragHandle" data-drag-handle="${id}" title="Перетащить">⋮⋮</span>
  <b>#${id}</b>
  <span class="blockTypeBadge">${BX.util.htmlspecialchars(type)}</span>
</div>


---

4. Добавь функцию сохранения нового порядка

Внутрь BX.ready(function () { ... }) добавь:

function saveBlockOrder() {
  const ids = Array.from(blocksBox.querySelectorAll('.block[data-block-id]'))
    .map(el => parseInt(el.getAttribute('data-block-id'), 10))
    .filter(Boolean);

  return api('block.reorder', {
    pageId,
    order: JSON.stringify(ids)
  });
}


---

5. Добавь drag-and-drop логику

Тоже внутрь BX.ready(function () { ... }) добавь:

function initBlockDnD() {
  if (!blocksBox) return;

  let draggedEl = null;

  blocksBox.querySelectorAll('.block[data-block-id]').forEach(block => {
    block.setAttribute('draggable', 'true');

    block.addEventListener('dragstart', (e) => {
      const handle = e.target.closest('[data-drag-handle]');
      if (!handle) {
        e.preventDefault();
        return;
      }

      draggedEl = block;
      block.classList.add('dragging');
      try { e.dataTransfer.effectAllowed = 'move'; } catch (err) {}
    });

    block.addEventListener('dragend', () => {
      if (draggedEl) draggedEl.classList.remove('dragging');
      draggedEl = null;
    });

    block.addEventListener('dragover', (e) => {
      if (!draggedEl || draggedEl === block) return;
      e.preventDefault();

      const rect = block.getBoundingClientRect();
      const middle = rect.top + rect.height / 2;
      const after = e.clientY > middle;

      if (after) {
        if (block.nextSibling !== draggedEl) {
          block.parentNode.insertBefore(draggedEl, block.nextSibling);
        }
      } else {
        if (block.previousSibling !== draggedEl) {
          block.parentNode.insertBefore(draggedEl, block);
        }
      }
    });

    block.addEventListener('drop', async (e) => {
      e.preventDefault();
      if (!draggedEl) return;

      try {
        const r = await saveBlockOrder();
        if (!r || r.ok !== true) {
          notify('Не удалось сохранить порядок');
          loadBlocks();
          return;
        }
        notify('Порядок сохранён');
        loadBlocks();
      } catch (err) {
        notify('Ошибка block.reorder');
        loadBlocks();
      }
    });
  });
}


---

6. После renderBlocks() нужно инициализировать DnD

В loadBlocks() замени:

renderBlocks(res.blocks);

на:

renderBlocks(res.blocks);
initBlockDnD();


---

Как проверить

Проверь так:

1. Открой страницу с несколькими блоками


2. Тяни блок за ⋮⋮


3. Отпусти выше или ниже другого блока


4. Должно:

визуально переставиться

показать Порядок сохранён

после перезагрузки страницы порядок должен остаться тем же





---

Если хочешь быстрее и чище

Я могу следующим сообщением прислать тебе полностью готовый editor.php уже со встроенным drag-and-drop, чтобы ты просто заменил файл целиком.