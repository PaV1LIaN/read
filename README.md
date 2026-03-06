Отлично.

Теперь делаем статус draft/published прямо в списке страниц и быстрое переключение без захода в настройки страницы.

Что добавим

В дереве страниц:

бейдж статуса,

кнопку Draft,

кнопку Published.



---

1. В index.php в renderPagesTree() покажи статус страницы

Найди кусок:

<div class="nodeMeta">
  <span>sort: ${parseInt(node.sort||500,10)}</span>
  <span>${parentLabel}</span>
</div>

И замени на:

<div class="nodeMeta">
  <span>sort: ${parseInt(node.sort||500,10)}</span>
  <span>${parentLabel}</span>
  <span class="nodeStatus ${String(node.status || 'published') === 'draft' ? 'isDraft' : 'isPublished'}">
    ${BX.util.htmlspecialchars(String(node.status || 'published').toUpperCase())}
  </span>
</div>


---

2. В CSS index.php добавь стили статуса

В <style> добавь:

.nodeStatus {
  display:inline-flex;
  align-items:center;
  padding:2px 8px;
  border-radius:999px;
  font-size:11px;
  font-weight:700;
  border:1px solid #e5e7ea;
}

.nodeStatus.isDraft {
  background:#fff7ed;
  border-color:#fdba74;
  color:#9a3412;
}

.nodeStatus.isPublished {
  background:#ecfdf3;
  border-color:#86efac;
  color:#166534;
}


---

3. В кнопки страницы добавь Draft / Published

Внутри .nodeBtns добавь две кнопки:

<button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-status="${node.id}" data-status="draft">Draft</button>
<button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-status="${node.id}" data-status="published">Published</button>

Полный фрагмент будет примерно таким:

<div class="nodeBtns">
  <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-move="${node.id}" data-dir="up">↑</button>
  <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-move="${node.id}" data-dir="down">↓</button>

  <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-parent="${node.id}">Вложить…</button>
  <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-root="${node.id}">В корень</button>

  <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-rename="${node.id}">Имя/slug</button>
  <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-duplicate="${node.id}">Дублировать</button>

  <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-status="${node.id}" data-status="draft">Draft</button>
  <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-status="${node.id}" data-status="published">Published</button>

  <a class="ui-btn ui-btn-primary ui-btn-xs btnTiny"
     href="/local/sitebuilder/editor.php?siteId=${siteId}&pageId=${node.id}"
     target="_blank">Редактор</a>

  <a class="ui-btn ui-btn-light ui-btn-xs btnTiny"
     href="/local/sitebuilder/view.php?siteId=${siteId}&pageId=${node.id}"
     target="_blank">Открыть</a>

  <button class="ui-btn ui-btn-danger ui-btn-xs btnTiny" data-page-delete="${node.id}">Удалить</button>
</div>


---

4. Добавь обработчик клика в index.php

Внутри container.addEventListener('click', function(e){ ... }) добавь:

const st = e.target.closest('[data-page-status]');
if (st) {
  const id = parseInt(st.getAttribute('data-page-status'), 10);
  const status = st.getAttribute('data-status') || 'draft';

  api('page.setStatus', { id, status }).then(r => {
    if (!r || r.ok !== true) {
      notify('Не удалось изменить статус');
      return;
    }
    notify('Статус изменён: ' + status);
    loadAndRender();
  }).catch(() => notify('Ошибка page.setStatus'));
  return;
}


---

Как проверить

1. Открой Страницы.


2. У страницы должен отображаться бейдж DRAFT или PUBLISHED.


3. Нажми Draft — статус должен стать DRAFT.


4. Нажми Published — статус должен стать PUBLISHED.


5. Обнови окно ещё раз и проверь, что статус сохранился.



Если это ок, следующим шагом я бы сделал скрытие draft-страниц из public.php, чтобы наружу никогда не уходили черновики.