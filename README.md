Да, это из-за ключа в localStorage.

Сейчас у тебя, скорее всего, состояние хранится по ключу:

const storageKey = 'sb-left-tree:' + location.pathname + location.search;

А location.search меняется на каждой странице, потому что меняется page=....
Из-за этого при переходе на вложенный пункт берётся новый пустой ключ, и ветка снова схлопывается.

Что исправить

Найди внизу public.php этот код:

const storageKey = 'sb-left-tree:' + location.pathname + location.search;

И замени на это:

const siteKey = document.body.getAttribute('data-site-key') || location.pathname;
const storageKey = 'sb-left-tree:' + siteKey;


---

Теперь нужно передать data-site-key в <body>

Найди:

<body style="--left-col: <?=h($leftCol)?>; --right-col: <?=h($rightCol)?>;">

Замени на:

<body
  data-site-key="<?=h((string)($site['slug'] ?? ('site-'.$siteId)))?>"
  style="--left-col: <?=h($leftCol)?>; --right-col: <?=h($rightCol)?>;"
>


---

Почему это решает проблему

Теперь состояние веток будет храниться:

не для каждой отдельной страницы,

а для всего сайта целиком.


То есть:

открыл ветку на одной странице,

перешёл на дочернюю,

ветка останется открытой.



---

Если хочешь ещё лучше

Можно сделать так, чтобы:

все предки активной страницы всегда принудительно были открыты,

даже если пользователь их раньше закрыл.


Но сначала внеси правку выше — скорее всего, этого уже хватит.