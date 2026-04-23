Причина уже понятна: у тебя bootstrap работает, но в views/blocks/disk.php сейчас рендерится только пустой контейнер:

<div class="sb-disk" ...></div>

А script.js рассчитан на полный HTML шаблон компонента — с:

toolbar

data-state="loading"

data-state="empty"

data-role="items-table"

data-view-container="grid"

и т.д.


То есть сейчас JS работает, но ему некуда рисовать интерфейс. Поэтому визуально ты видишь пустой белый блок.


---

Что нужно сделать

Правильный фикс

Вместо голого контейнера в:

/local/sitebuilder/views/blocks/disk.php

нужно рендерить сам компонент целиком.


---

Замени /local/sitebuilder/views/blocks/disk.php на это

<?php
$site = $context['site'] ?? [];
$currentPage = $context['currentPage'] ?? [];
$siteId = (int)($site['id'] ?? 0);
$pageId = (int)($currentPage['id'] ?? 0);
$blockId = (int)($block['id'] ?? 0);

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/components/disk/class.php';

$component = new SitebuilderDiskComponent([
    'SITE_ID' => $siteId,
    'PAGE_ID' => $pageId,
    'BLOCK_ID' => $blockId,
    'CURRENT_USER_ID' => (int)$USER->GetID(),
]);

$component->execute();


---

Почему это нужно

Потому что файл:

/local/sitebuilder/components/disk/template.php

уже содержит весь нужный HTML интерфейса.

А сейчас ты вывел только оболочку, без разметки состояний и таблицы.


---

Что проверить после замены

1. сохрани views/blocks/disk.php


2. обнови страницу Ctrl + F5


3. снова открой публичную страницу




---

Что должно измениться

Если папка пустая, ты теперь увидишь:

шапку блока

toolbar

пустое состояние: «Здесь пока нет файлов и папок.»


Если в папке есть элементы — увидишь список.


---

Важный момент

Чтобы этот код сработал, в views/blocks/disk.php должен быть доступен global $USER.

На всякий случай можно сделать так:

<?php
global $USER;

$site = $context['site'] ?? [];
$currentPage = $context['currentPage'] ?? [];
$siteId = (int)($site['id'] ?? 0);
$pageId = (int)($currentPage['id'] ?? 0);
$blockId = (int)($block['id'] ?? 0);

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/components/disk/class.php';

$component = new SitebuilderDiskComponent([
    'SITE_ID' => $siteId,
    'PAGE_ID' => $pageId,
    'BLOCK_ID' => $blockId,
    'CURRENT_USER_ID' => (int)$USER->GetID(),
]);

$component->execute();


---

После этого, если что-то еще останется пустым, уже нужно будет смотреть list response, но сейчас главная проблема именно в том, что ты рендерил не шаблон компонента, а только пустой div.