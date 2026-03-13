Да, причина именно в текущей раскладке pageShell.

Сейчас центр у тебя участвует в одной общей grid вместе с left/right, поэтому:

включаешь left

слева появляется колонка

центр математически сдвигается вправо


То есть это не баг контента, а баг CSS-архитектуры layout.

Почему так происходит

У тебя, скорее всего, сейчас что-то вроде этого:

.pageShell{
  display:grid;
  grid-template-columns: var(--left-col) minmax(0,1fr) minmax(0,var(--sb-container)) minmax(0,1fr) var(--right-col);
}

Из-за этого центральная колонка центрируется вместе с боковыми, а не сама по себе.


---

Как исправить правильно

Нужно сделать так, чтобы:

центр всегда был по центру экрана

left и right жили отдельно от него

боковые колонки не влияли на позицию центра



---

Что поменять в public.php

1. Замени стили pageShell / pageLeft / pageCenter / pageRight

Найди текущие стили:

.pageShell
.pageLeft
.pageCenter
.pageRight

И замени их целиком на это:

.pageShell{
  position:relative;
  width:100%;
  padding:24px 16px 40px;
}

.pageCenter{
  width:min(var(--sb-container), calc(100% - 32px));
  margin:0 auto;
  min-width:0;
}

.pageLeft{
  position:absolute;
  left:16px;
  top:24px;
  width:var(--left-col);
  min-width:0;
}

.pageRight{
  position:absolute;
  right:16px;
  top:24px;
  width:var(--right-col);
  min-width:0;
}


---

2. Подправь sidebar box

Найди:

.layoutSidebarBox{
  ...
  position: sticky;
  top: 88px;
}

Оставь sticky, но добавь ширину:

.layoutSidebarBox{
  background:#fff;
  border:1px solid #eef2f6;
  border-radius:14px;
  padding:12px;
  box-shadow:0 1px 2px rgba(0,0,0,.03);
  position:sticky;
  top:88px;
  width:100%;
}


---

3. Обнови media query

Найди текущий блок:

@media (max-width: 1200px){
  .pageShell{
    grid-template-columns:1fr;
    padding:24px 16px 40px;
  }
  .pageLeft, .pageCenter, .pageRight{
    grid-column:1;
  }
  .layoutSidebarBox{
    position:static;
  }
}

И замени на:

@media (max-width: 1200px){
  .pageShell{
    padding:24px 16px 40px;
  }

  .pageLeft,
  .pageRight,
  .pageCenter{
    position:static;
    width:100%;
    margin:0 0 16px 0;
  }

  .layoutSidebarBox{
    position:static;
  }
}


---

Что это даст

Теперь:

центр всегда останется по центру

включение left больше не будет сдвигать центральную колонку

right тоже не будет влиять на центр

на узких экранах всё снова сложится в одну колонку



---

Важно

У такого решения есть нюанс:

если leftWidth/rightWidth слишком большие

и экран не очень широкий, то боковые колонки могут визуально приближаться к центру


Но это уже нормальное поведение. Главная проблема со смещением центра уйдёт.


---

Если хочешь совсем аккуратный вариант

Следующим шагом можно сделать ещё лучше:

автоматически уменьшать left/right, если не хватает места

или скрывать одну из колонок раньше

или делать “прилипание” боковых колонок к центру, а не к краям экрана


Но сначала внеси этот CSS — он решает именно твою текущую проблему.