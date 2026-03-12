Да, теперь видно точное место.
Список страниц действительно нужно привести в порядок именно через renderPagesTree(...) и CSS.

Я бы сделал так:

карточка страницы в 3 строки

слева информация, справа действия

мету упростить

кнопки сгруппировать

дерево оставить, но сделать аккуратнее


Ниже даю готовые правки.


---

1. Замени CSS для дерева страниц

В <style> найди старые стили блока страниц:

.tree

.node

.nodeHead

.nodeLeft

.nodeIcon

.nodeMain

.nodeTitleLine

.nodeSlug

.nodeMeta

.nodeBtns

.children

.btnTiny

.nodeBadges

.pageBadge...

.node.isDraft

.nodeMeta code


И замени их целиком на этот блок:

.tree{
  margin-top:12px;
}

.node{
  border:1px solid #e7ebf0;
  border-radius:14px;
  padding:12px;
  background:#fff;
  margin-top:10px;
  box-shadow:0 1px 2px rgba(0,0,0,.03);
}

.node.isDraft{
  background:#fffaf5;
  border-color:#f2d4b3;
}

.nodeHead{
  display:grid;
  grid-template-columns:minmax(0,1fr) auto;
  gap:12px;
  align-items:start;
}

.nodeLeft{
  display:flex;
  gap:10px;
  align-items:flex-start;
  min-width:0;
}

.nodeIcon{
  width:24px;
  height:24px;
  border-radius:8px;
  background:#f3f4f6;
  color:#6b7280;
  display:flex;
  align-items:center;
  justify-content:center;
  flex:0 0 auto;
  font-size:12px;
  font-weight:700;
}

.nodeMain{
  min-width:0;
  display:flex;
  flex-direction:column;
  gap:6px;
}

.nodeTitleLine{
  display:flex;
  align-items:center;
  gap:8px;
  flex-wrap:wrap;
}

.nodeTitle{
  font-size:15px;
  font-weight:700;
  line-height:1.25;
  color:#111827;
}

.nodeSlug{
  display:inline-flex;
  align-items:center;
  padding:2px 8px;
  border-radius:999px;
  font-size:11px;
  background:#f3f4f6;
  color:#4b5563;
  border:1px solid #eceff3;
}

.nodeBadges{
  display:flex;
  gap:6px;
  flex-wrap:wrap;
  align-items:center;
}

.pageBadge{
  display:inline-flex;
  align-items:center;
  gap:6px;
  padding:3px 8px;
  border-radius:999px;
  font-size:11px;
  font-weight:700;
  line-height:1;
  border:1px solid transparent;
}

.pageBadgePublished{
  background:#ecfdf3;
  border-color:#b7ebc6;
  color:#027a48;
}

.pageBadgeDraft{
  background:#fff4ed;
  border-color:#ffd6ae;
  color:#b54708;
}

.pageBadgeHome{
  background:#eef2ff;
  border-color:#c7d2fe;
  color:#3730a3;
}

.nodeMeta{
  display:flex;
  gap:8px;
  flex-wrap:wrap;
  font-size:12px;
  color:#6b7280;
}

.nodeMetaItem{
  display:inline-flex;
  align-items:center;
  gap:4px;
  padding:2px 8px;
  border-radius:999px;
  background:#f8fafc;
  border:1px solid #eef2f6;
}

.nodeMeta code{
  background:#f3f4f6;
  padding:1px 6px;
  border-radius:999px;
  font-size:11px;
}

.nodeBtns{
  display:flex;
  flex-wrap:wrap;
  gap:6px;
  align-items:center;
  justify-content:flex-end;
  max-width:380px;
}

.children{
  margin-left:16px;
  border-left:2px solid #eef2f6;
  padding-left:12px;
  margin-top:10px;
}

.btnTiny{
  padding:0 8px;
  height:28px;
  line-height:28px;
  border-radius:8px;
}

.pageActionsGroup{
  display:flex;
  gap:6px;
  flex-wrap:wrap;
  align-items:center;
}

@media (max-width: 900px){
  .nodeHead{
    grid-template-columns:1fr;
  }

  .nodeBtns{
    justify-content:flex-start;
    max-width:none;
  }
}


---

2. Замени HTML карточки страницы в renderPagesTree

Найди внутри renderNode вот этот кусок:

return `
  <div class="node ${isDraft ? 'isDraft' : ''}">
      ...
  </div>
`;

И замени целиком весь return на это:

return `
  <div class="node ${isDraft ? 'isDraft' : ''}">
    <div class="nodeHead">
      <div class="nodeLeft">
        <div class="nodeIcon">≡</div>

        <div class="nodeMain">
          <div class="nodeTitleLine">
            <div class="nodeTitle">#${node.id} ${title}</div>
            <span class="nodeSlug">${slug}</span>
          </div>

          <div class="nodeBadges">
            ${statusBadge}
            ${homeBadge}
          </div>

          <div class="nodeMeta">
            <span class="nodeMetaItem">sort <code>${sort}</code></span>
            <span class="nodeMetaItem">${parentLabel}</span>
            <span class="nodeMetaItem">slug <code>${slug}</code></span>
          </div>
        </div>
      </div>

      <div class="nodeBtns">
        <div class="pageActionsGroup">
          <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-move="${node.id}" data-dir="up">↑</button>
          <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-move="${node.id}" data-dir="down">↓</button>
          <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-parent="${node.id}">Вложить…</button>
          <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-root="${node.id}">В корень</button>
        </div>

        <div class="pageActionsGroup">
          <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-rename="${node.id}">Имя/slug</button>

          <button class="ui-btn ${draftBtnClass} ui-btn-xs btnTiny" data-page-status="${node.id}" data-status="draft">Draft</button>
          <button class="ui-btn ${pubBtnClass} ui-btn-xs btnTiny" data-page-status="${node.id}" data-status="published">Published</button>
        </div>

        <div class="pageActionsGroup">
          <a class="ui-btn ui-btn-primary ui-btn-xs btnTiny"
             href="/local/sitebuilder/editor.php?siteId=${siteId}&pageId=${node.id}"
             target="_blank">Редактор</a>

          <a class="ui-btn ui-btn-light ui-btn-xs btnTiny"
             href="/local/sitebuilder/view.php?siteId=${siteId}&pageId=${node.id}"
             target="_blank">Открыть</a>

          <button class="ui-btn ui-btn-danger ui-btn-xs btnTiny" data-page-delete="${node.id}">Удалить</button>
        </div>
      </div>
    </div>

    ${hasVisibleKids ? `<div class="children">${kidsHtml}</div>` : ''}
  </div>
`;


---

3. Упрости parentLabel

Сейчас у тебя:

const parentLabel = pid ? `parent: #${pid}` : 'root';

Замени на:

const parentLabel = pid ? `parent #${pid}` : 'root';

Так будет визуально чище.


---

4. Небольшая правка заголовка дерева

Сейчас сверху в диалоге есть строка:

<div class="hint2">
  Дерево: <code>parentId</code>. Порядок: <code>sort</code> (стрелки ↑/↓ меняют порядок среди “соседей”).
</div>

Я бы заменил на более человеческую:

<div class="hint2">
  Страницы выводятся деревом по <code>parentId</code>. Стрелки ↑/↓ меняют порядок среди соседних страниц одного уровня.
</div>


---

Что получится

После этого:

каждая страница будет выглядеть как нормальная карточка

дерево останется

статус и HOME будут читаться лучше

кнопки не будут выглядеть как хаос

карточки станут удобнее для дальнейшего расширения



---

Что я бы делал сразу после этого

После нормализации списка страниц уже можно спокойно идти к section / content areas.
Потому что админка страниц и навигация будут уже не стыдными.