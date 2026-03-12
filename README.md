Дальше я бы добавил быстрые переключатели статуса прямо в списке страниц:

кнопка Draft

кнопка Published

без захода внутрь страницы


Это как раз логичное продолжение после улучшения списка страниц.

Что получится

В дереве страниц у каждой строки будут:

Draft

Published

текущий статус будет сразу виден

переключение в один клик


Для этого нужно сделать 2 вещи

1. В index.php добавить кнопки в renderPagesTree()

В блоке nodeBtns рядом с Имя/slug вставь:

const draftBtnClass = isDraft ? 'ui-btn-warning' : 'ui-btn-light';
const pubBtnClass = !isDraft ? 'ui-btn-success' : 'ui-btn-light';

И внутри кнопок:

<button class="ui-btn ${draftBtnClass} ui-btn-xs btnTiny" data-page-status="${node.id}" data-status="draft">Draft</button>
<button class="ui-btn ${pubBtnClass} ui-btn-xs btnTiny" data-page-status="${node.id}" data-status="published">Published</button>

То есть кусок nodeBtns станет таким:

<div class="nodeBtns">
  <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-move="${node.id}" data-dir="up">↑</button>
  <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-move="${node.id}" data-dir="down">↓</button>

  <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-parent="${node.id}">Вложить…</button>
  <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-root="${node.id}">В корень</button>

  <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-rename="${node.id}">Имя/slug</button>

  <button class="ui-btn ${draftBtnClass} ui-btn-xs btnTiny" data-page-status="${node.id}" data-status="draft">Draft</button>
  <button class="ui-btn ${pubBtnClass} ui-btn-xs btnTiny" data-page-status="${node.id}" data-status="published">Published</button>

  <a class="ui-btn ui-btn-primary ui-btn-xs btnTiny"
     href="/local/sitebuilder/editor.php?siteId=${siteId}&pageId=${node.id}"
     target="_blank">Редактор</a>

  <a class="ui-btn ui-btn-light ui-btn-xs btnTiny"
     href="/local/sitebuilder/view.php?siteId=${siteId}&pageId=${node.id}"
     target="_blank">Открыть</a>

  <button class="ui-btn ui-btn-danger ui-btn-xs btnTiny" data-page-delete="${node.id}">Удалить</button>
</div>


---

2. В обработчик кликов внутри openPagesDialog() добавить смену статуса

В container.addEventListener('click', function(e){ ... }) добавь такой блок до удаления страницы:

const st = e.target.closest('[data-page-status]');
if (st) {
  const id = parseInt(st.getAttribute('data-page-status'), 10);
  const status = st.getAttribute('data-status') || 'draft';

  api('page.setStatus', { id, status }).then(r => {
    if (!r || r.ok !== true) { notify('Не удалось сменить статус'); return; }
    loadAndRender();
  }).catch(() => notify('Ошибка page.setStatus'));
  return;
}


---

3. В api.php нужен action page.setStatus

Если его ещё нет, добавь:

if ($action === 'page.setStatus') {
    $id = (int)($_POST['id'] ?? 0);
    $status = strtolower(trim((string)($_POST['status'] ?? 'draft')));

    if ($id <= 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    if (!in_array($status, ['draft', 'published'], true)) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'BAD_STATUS'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $page = sb_find_page($id);
    if (!$page) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $pages = sb_read_pages();
    $found = false;

    foreach ($pages as &$p) {
        if ((int)($p['id'] ?? 0) === $id) {
            $p['status'] = $status;
            $p['updatedAt'] = date('c');
            $p['updatedBy'] = (int)$USER->GetID();
            $found = true;
            break;
        }
    }
    unset($p);

    if (!$found) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    sb_write_pages($pages);
    echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE);
    exit;
}


---

После этого в списке страниц можно будет щёлкать Draft / Published прямо на месте.

Если хочешь, следующим сообщением я могу сразу собрать тебе готовый кусок renderPagesTree() целиком с уже встроенными этими кнопками.