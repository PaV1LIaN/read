Да, причина уже не в сдвиге, а в том, что тексту разрешён слишком агрессивный перенос в узком sidebar.

У тебя сейчас слова режутся из-за этого стиля:

word-break: break-word;

или похожего поведения в слишком узкой колонке.

Что исправить

1. Замени стиль .sideMenuText

Найди:

.sideMenuText{
  min-width:0;
  flex:1 1 auto;
  word-break:break-word;
}

И замени на:

.sideMenuText{
  min-width:0;
  flex:1 1 auto;
  word-break:normal;
  overflow-wrap:anywhere;
}

Это уже сильно улучшит перенос.


---

2. Сделай дочерние уровни ещё компактнее

Найди:

.sideTreeChildren{
  margin-top:2px;
  margin-left:6px;
  padding-left:6px;
  border-left:1px solid #f1f5f9;
}

И замени на:

.sideTreeChildren{
  margin-top:2px;
  margin-left:2px;
  padding-left:6px;
  border-left:1px solid #f1f5f9;
}


---

3. Убери лишнюю ширину у стрелки/заглушки почти в ноль

Найди:

.sideToggle{
  width:14px;
  height:18px;
  flex:0 0 14px;

Замени на:

.sideToggle{
  width:12px;
  height:18px;
  flex:0 0 12px;

И так же:

.sideToggleStub{
  width:14px;
  height:18px;
  flex:0 0 14px;

Замени на:

.sideToggleStub{
  width:12px;
  height:18px;
  flex:0 0 12px;


---

4. Самое важное: увеличь leftWidth

По скрину видно, что глубина уже 4–5 уровней, а колонка слишком узкая.
Для такого дерева leftWidth в 220–230 уже мало.

Поставь в настройках layout:

leftWidth = 280

или даже

leftWidth = 300

Именно это добьёт проблему.
При 5 уровнях вложенности узкий sidebar почти неизбежно начнёт ломать текст.


---

Если хочешь прямо оптимальный вариант

Для текущей структуры я бы поставил:

leftWidth: 300

минимальные отступы как выше

word-break: normal

overflow-wrap: anywhere



---

Почему сейчас всё ещё ломается

Потому что даже при уменьшенном сдвиге у тебя:

много уровней вложенности

слева остаются линии и отступы

ширина колонки ограничена

браузер вынужден переносить слово буквально по буквам


То есть тут уже нужен не только “меньший сдвиг”, но и чуть более широкая левая колонка.

Если хочешь, я следующим сообщением дам тебе готовый финальный CSS-блок для left menu целиком, чтобы ты просто заменил всё сразу.