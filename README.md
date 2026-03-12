Да, давай попробуем правильно: убираем карточки и делаем компактное дерево строк, чтобы большая вложенность не ломала вёрстку.

Это для списка страниц подходит намного лучше.


---

1. Замени CSS дерева страниц

Найди текущие стили:

.tree

.node

.nodeHead

.nodeLeft

.nodeIcon

.nodeMain

.nodeTitleLine

.nodeTitle

.nodeSlug

.nodeBadges

.pageBadge...

.nodeMeta

.nodeMetaItem

.nodeBtns

.children

.btnTiny

.pageActionsGroup


И замени их целиком на это:

.tree{
  margin-top:12px;
}

.node{
  margin-top:8px;
}

.nodeRow{
  display:flex;
  align-items:flex-start;
  justify-content:space-between;
  gap:12px;
  padding:10px 12px;
  border:1px solid #e7ebf0;
  border-radius:12px;
  background:#fff;
}

.node.isDraft .nodeRow{
  background:#fffaf5;
  border-color:#f2d4b3;
}

.nodeLeft{
  display:flex;
  gap:10px;
  align-items:flex-start;
  min-width:0;
  flex:1 1 auto;
}

.nodeIcon{
  width:22px;
  height:22px;
  border-radius:8px;
  background:#f3f4f6;
  color:#6b7280;
  display:flex;
  align-items:center;
  justify-content:center;
  flex:0 0 auto;
  font-size:11px;
  font-weight:700;
}

.nodeMain{
  min-width:0;
  flex:1 1 auto;
  display:flex;
  flex-direction:column;
  gap:4px;
}

.nodeTitleLine{
  display:flex;
  align-items:center;
  gap:8px;
  flex-wrap:wrap;
  min-width:0;
}

.nodeTitle{
  font-size:14px;
  font-weight:700;
  line-height:1.25;
  color:#111827;
  word-break:break-word;
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
  flex:0 0 360px;
}

.btnTiny{
  padding:0 8px;
  height:28px;
  line-height:28px;
  border-radius:8px;
}

.children{
  margin-left:14px;
  padding-left:10px;
  border-left:1px solid #eef2f6;
  margin-top:6px;
}

@media (max-width: 1100px){
  .nodeRow{
    flex-direction:column;
    align-items:stretch;
  }

  .nodeBtns{
    flex:1 1 auto;
    justify-content:flex-start;
  }
}


---

2. Замени HTML в renderPagesTree

Найди внутри renderNode(node) текущий return и замени целиком на это:

return `
  <div class="node ${isDraft ? 'isDraft' : ''}">
    <div class="nodeRow">
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
          </div>
        </div>
      </div>

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
    </div>

    ${hasVisibleKids ? `<div class="children">${kidsHtml}</div>` : ''}
  </div>
`;


---

3. Упрости parentLabel

Если у тебя сейчас не так, сделай:

const parentLabel = pid ? `parent #${pid}` : 'root';


---

Что получится

Теперь вместо:

карточка внутри карточки

текст и кнопки налезают

глубокие уровни ломаются


будет:

компактная строка

дерево остаётся

вложенность читается

большая глубина переносится намного лучше



---

После того как проверишь, следующим шагом уже можно спокойно идти к content sections / section block.