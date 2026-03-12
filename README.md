Да, доведём.

Сейчас цель — сделать левое меню более похожим на нормальное дерево навигации, а не на набор вложенных карточек.

Что меняем

Сделаем:

меньше рамок

спокойнее active

чище вложенность

компактнее уровни

визуальный акцент не на “капсуле”, а на текущей ветке



---

1. Замени стили левого меню

В public.php найди текущие стили:

.sideTree

.sideTreeChildren

.sideTreeNode

.sideMenuLink

.sideMenuLink:hover

.sideMenuLink.active

.sideMenuLink.open

.sideMenuCaret

.sideMenuCaretEmpty

.sideMenuText


И полностью замени их на это:

.sideTree{
  display:flex;
  flex-direction:column;
  gap:2px;
}

.sideTreeNode{
  min-width:0;
}

.sideTreeChildren{
  margin-top:2px;
  margin-left:12px;
  padding-left:12px;
  border-left:1px solid #eceff3;
}

.sideMenuLink{
  display:flex;
  align-items:flex-start;
  gap:8px;
  width:100%;
  padding:6px 8px;
  border-radius:8px;
  border:1px solid transparent;
  background:transparent;
  color:var(--text);
  text-decoration:none;
  line-height:1.35;
  transition:background .15s ease, color .15s ease, border-color .15s ease;
}

.sideMenuLink:hover{
  text-decoration:none;
  background:#f8fafc;
  color:var(--text);
}

.sideMenuLink.open{
  background:transparent;
}

.sideMenuLink.active{
  background:#f5f9ff;
  border-color:#dbeafe;
  color:#1d4ed8;
  font-weight:600;
}

.sideMenuCaret{
  width:12px;
  flex:0 0 12px;
  color:#94a3b8;
  text-align:center;
  line-height:1.2;
  font-size:10px;
  margin-top:2px;
}

.sideMenuCaretEmpty{
  visibility:hidden;
}

.sideMenuText{
  min-width:0;
  flex:1 1 auto;
  word-break:break-word;
}

.sideTree.level-0 > .sideTreeNode > .sideMenuLink{
  font-weight:600;
  padding-top:7px;
  padding-bottom:7px;
}

.sideTree.level-1 > .sideTreeNode > .sideMenuLink,
.sideTree.level-2 > .sideTreeNode > .sideMenuLink,
.sideTree.level-3 > .sideTreeNode > .sideMenuLink,
.sideTree.level-4 > .sideTreeNode > .sideMenuLink{
  font-weight:400;
  font-size:14px;
}


---

2. Сделай сам sidebar чуть аккуратнее

Найди стиль:

.layoutSidebarBox

И замени на:

.layoutSidebarBox{
  background:#fff;
  border:1px solid #eef2f6;
  border-radius:14px;
  padding:12px;
  box-shadow: 0 1px 2px rgba(0,0,0,.03);
  position: sticky;
  top: 88px;
}


---

3. Если хочешь ещё спокойнее — уменьши ширину left

Сейчас у тебя, скорее всего, leftWidth = 260.
Для такого дерева я бы попробовал:

220

или

230

Это обычно выглядит аккуратнее.


---

4. Небольшое улучшение breadcrumbs

Чтобы они не спорили с заголовком, можешь сделать их чуть мягче.

Найди стиль .breadcrumbs и замени на:

.breadcrumbs{
  display:flex;
  flex-wrap:wrap;
  align-items:center;
  gap:6px;
  margin-bottom:8px;
  font-size:12px;
  color:#94a3b8;
}

И .crumb.current:

.crumb.current{
  color:#64748b;
  font-weight:500;
}


---

Что получится

После этого:

меню станет менее “пузырчатым”

активная страница останется заметной, но не будет выглядеть как кнопка внутри кнопки

дерево станет ближе к нормальному сайдбару CMS



---

Что я бы делал следующим шагом

После этой косметики уже логично сделать раскрытие/сворачивание веток по клику на стрелку.