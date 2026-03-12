Да, это легко.

В public.php уменьши размер шрифта у левого меню.

Что поменять

1. Найди стиль .sideMenuLink

Сейчас там, скорее всего, нет явного font-size или он наследуется.

Сделай так:

.sideMenuLink{
  display:flex;
  align-items:flex-start;
  gap:8px;
  width:100%;
  padding:6px 6px;
  border-radius:8px;
  border:1px solid transparent;
  background:transparent;
  color:var(--text);
  text-decoration:none;
  line-height:1.3;
  font-size:13px;
  transition:background .15s ease, color .15s ease, border-color .15s ease;
}


---

2. Для верхнего уровня тоже чуть уменьшить

Найди:

.sideTree.level-0 > .sideTreeNode > .sideMenuLink{
  font-weight:700;
}

Замени на:

.sideTree.level-0 > .sideTreeNode > .sideMenuLink{
  font-weight:700;
  font-size:13px;
}


---

3. Для вложенных уровней ещё компактнее

Найди блок:

.sideTree.level-1 > .sideTreeNode > .sideMenuLink,
.sideTree.level-2 > .sideTreeNode > .sideMenuLink,
.sideTree.level-3 > .sideTreeNode > .sideMenuLink,
.sideTree.level-4 > .sideTreeNode > .sideMenuLink{
  font-weight:400;
  font-size:14px;
}

И замени на:

.sideTree.level-1 > .sideTreeNode > .sideMenuLink,
.sideTree.level-2 > .sideTreeNode > .sideMenuLink,
.sideTree.level-3 > .sideTreeNode > .sideMenuLink,
.sideTree.level-4 > .sideTreeNode > .sideMenuLink,
.sideTree.level-5 > .sideTreeNode > .sideMenuLink{
  font-weight:400;
  font-size:12px;
}


---

Если хочешь совсем аккуратно

Можно сделать единый вариант:

верхний уровень 13px

все вложенные 12px


Это обычно хорошо смотрится в sidebar.

Если после этого захочешь, следующим шагом можно ещё немного уменьшить line-height, чтобы меню стало компактнее по высоте.