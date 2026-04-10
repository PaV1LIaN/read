Да, согласен — текущее левое меню выглядит слишком “технически”:

кнопки раскрытия живут отдельно от ссылки

вложенность читается, но визуально тяжёлая

много маленьких рамок внутри рамок

дерево похоже на debug-ui, а не на нормальную навигацию раздела


Что лучше сделать

Я бы привёл левое меню к такому виду:

одна общая карточка раздела

внутри — чистое дерево

toggle компактный и встроенный в строку

активный пункт выделен мягко

вложенность читается за счёт отступов и тонкой вертикальной линии

без лишних отдельных коробок на каждом уровне


То есть не “кнопка + кнопка + карточка”, а именно sidebar tree navigation.


---

Что предлагаю

Сейчас не трогаем PHP-логику дерева, только меняем визуальную подачу.

Нужно заменить стили дерева в:

/local/sitebuilder/assets/public/public.css


---

Что заменить в public.css

Найди и удали/замени текущие стили, связанные с:

.sb-section-nav__tree

.sb-tree-node

.sb-tree-node__row

.sb-tree-node__toggle

.sb-tree-node__toggle--empty

.sb-tree-node__toggle-icon

.sb-tree-node.is-open > .sb-tree-node__row .sb-tree-node__toggle-icon

.sb-tree-node__children

.sb-tree-node.is-open > .sb-tree-node__children

.sb-section-nav__link

.sb-section-nav__link.is-active

.sb-section-nav__text


и вставь вместо них вот это:

.sb-section-nav {
    display: flex;
    flex-direction: column;
    gap: 12px;
}

.sb-section-nav__title {
    font-size: 13px;
    font-weight: 700;
    color: #111827;
    letter-spacing: 0.01em;
}

.sb-section-nav__tree {
    display: flex;
    flex-direction: column;
    gap: 4px;
}

.sb-tree-node {
    display: flex;
    flex-direction: column;
    gap: 4px;
}

.sb-tree-node__row {
    display: flex;
    align-items: center;
    gap: 8px;
    padding-left: calc(var(--sb-nav-depth, 0) * 18px);
    position: relative;
}

.sb-tree-node__row::before {
    content: "";
    position: absolute;
    left: calc((var(--sb-nav-depth, 0) * 18px) - 9px);
    top: -4px;
    bottom: -4px;
    width: 1px;
    background: transparent;
}

.sb-tree-node[style*="--sb-nav-depth:1"] > .sb-tree-node__row::before,
.sb-tree-node[style*="--sb-nav-depth:2"] > .sb-tree-node__row::before,
.sb-tree-node[style*="--sb-nav-depth:3"] > .sb-tree-node__row::before,
.sb-tree-node[style*="--sb-nav-depth:4"] > .sb-tree-node__row::before,
.sb-tree-node[style*="--sb-nav-depth:5"] > .sb-tree-node__row::before {
    background: #e5e7eb;
}

.sb-tree-node__toggle {
    width: 20px;
    min-width: 20px;
    height: 20px;
    border: 0;
    border-radius: 6px;
    background: transparent;
    cursor: pointer;
    position: relative;
    padding: 0;
    flex: 0 0 20px;
}

.sb-tree-node__toggle:hover {
    background: #f3f4f6;
}

.sb-tree-node__toggle--empty {
    cursor: default;
    pointer-events: none;
}

.sb-tree-node__toggle-icon {
    position: absolute;
    top: 50%;
    left: 50%;
    width: 7px;
    height: 7px;
    border-right: 2px solid #64748b;
    border-bottom: 2px solid #64748b;
    transform: translate(-50%, -60%) rotate(45deg);
    transition: transform .15s ease;
}

.sb-tree-node.is-open > .sb-tree-node__row .sb-tree-node__toggle-icon {
    transform: translate(-50%, -40%) rotate(225deg);
}

.sb-tree-node__children {
    display: none;
    flex-direction: column;
    gap: 4px;
}

.sb-tree-node.is-open > .sb-tree-node__children {
    display: flex;
}

.sb-section-nav__link {
    display: flex;
    align-items: center;
    min-height: 36px;
    width: 100%;
    text-decoration: none;
    color: #374151;
    padding: 8px 12px;
    border-radius: 10px;
    background: transparent;
    border: 1px solid transparent;
    transition: background .15s ease, border-color .15s ease, color .15s ease;
}

.sb-section-nav__link:hover {
    background: #f8fafc;
    border-color: #e5e7eb;
}

.sb-section-nav__link.is-active {
    color: var(--sb-accent);
    background: #eef2ff;
    border-color: #c7d2fe;
    font-weight: 600;
}

.sb-section-nav__text {
    display: inline-block;
    font-size: 14px;
    line-height: 1.35;
}


---

Что это изменит

После замены меню станет:

визуально легче

ближе к нормальному sidebar tree

без ощущения “таблицы внутри таблицы”

вложенность будет видна через отступы и тонкие линии

активный пункт останется заметным

toggle станет второстепенным, а не главным элементом



---

Если хочешь ещё лучше

Следующий шаг, который я бы сделал после этого:

1. убрать заголовок раздела внутри левой карточки, если он дублирует текущую страницу


2. сделать левую колонку уже, а центральную шире


3. добавить sticky sidebar, если меню длинное


4. показывать только children root-а по умолчанию, а не весь root целиком, если корневая страница и так уже отображается в breadcrumbs и h1



Сейчас сначала лучше просто применить новые стили и посмотреть визуально.

Если хочешь, следующим сообщением я дам ещё более аккуратный вариант: сделать левое меню похожим на Bitrix/Notion-style sidebar, где активная ветка читается ещё лучше.