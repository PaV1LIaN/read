Да, вижу проблему: на глубоких уровнях текст уезжает слишком вправо и начинает ломаться в узкой колонке.

Нужно сделать вложенность визуально читаемой, но с минимальным сдвигом.

Что поменять

1. Уменьши отступ дочерних веток

Найди стиль:

.sideTreeChildren{
  margin-top:2px;
  margin-left:12px;
  padding-left:12px;
  border-left:1px solid #f1f5f9;
}

И замени на:

.sideTreeChildren{
  margin-top:2px;
  margin-left:6px;
  padding-left:6px;
  border-left:1px solid #f1f5f9;
}


---

2. Уменьши ширину зоны стрелки

Найди:

.sideToggle{
  width:18px;
  height:18px;
  flex:0 0 18px;

И замени на:

.sideToggle{
  width:14px;
  height:18px;
  flex:0 0 14px;


---

3. Так же уменьши заглушку

Найди:

.sideToggleStub{
  width:18px;
  height:18px;
  flex:0 0 18px;

И замени на:

.sideToggleStub{
  width:14px;
  height:18px;
  flex:0 0 14px;


---

4. Сделай расстояние между стрелкой и текстом меньше

Найди:

.sideMenuRow{
  display:flex;
  align-items:flex-start;
  gap:6px;
  min-width:0;
}

И замени на:

.sideMenuRow{
  display:flex;
  align-items:flex-start;
  gap:4px;
  min-width:0;
}


---

Если хочешь ещё аккуратнее

Можно ещё слегка уменьшить внутренние отступы ссылки:

Было:

.sideMenuLink{
  padding:6px 8px;
}

Сделай:

.sideMenuLink{
  padding:6px 6px;
}


---

Что получится

После этого:

каждый следующий уровень будет смещаться совсем немного

текст перестанет так быстро ломаться в столбик

дерево останется читаемым


Если хочешь, следующим сообщением я дам тебе уже готовый финальный CSS-блок для всего левого меню, чтобы ты просто заменил его целиком.