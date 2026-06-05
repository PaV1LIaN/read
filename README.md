Готово. Я взял рабочие файлы из твоего архива и точечно добавил туда секции/колонки, не переписывая весь визуал.

Скачай архив:

Скачать готовые файлы

Внутри 3 файла:

public_render.php
public_page.php
public.css

Замени ими файлы на сервере:

/local/sitebuilder/lib/public_render.php
/local/sitebuilder/views/layout/public_page.php
/local/sitebuilder/assets/public/public.css

После замены сделай:

php -l /srv/bx/docroot/local/sitebuilder/lib/public_render.php
php -l /srv/bx/docroot/local/sitebuilder/views/layout/public_page.php

Если ошибок нет — открой публичную страницу с обновлением:

Ctrl + F5

Я сделал так, чтобы:

1. public_page.php остался на базе твоей рабочей версии.
2. Визуал шапки, меню, фона и layout не переписывался.
3. Секции выводились через sb_public_render_page_sections().
4. Колонки задавались inline-стилем, чтобы CSS точно не мешал.
5. Текстовые блоки снова нормально читали content JSON.
6. Секции ищутся через pageSections, PageSectionRepository, DB и page_sections.json.

Если после замены в Ctrl + U найдётся:

sb-page-section__grid

значит публичка уже использует секции.