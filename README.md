Проблема найдена: в твоём urlrewrite.php нет нашего правила.

Нужно вставить его вот сюда — сразу после строки:

$arUrlRewrite=array (

То есть начало файла должно стать таким:

<?php
$arUrlRewrite=array (
  999001 =>
  array (
    'CONDITION' => '#^/local/mvc/?(.*)$#',
    'RULE' => 'route=/$1',
    'ID' => '',
    'PATH' => '/local/mvc/index.php',
    'SORT' => 1,
  ),

  4 => 
  array (
    'CONDITION' => '#^/docs/pub/(?<hash>[0-9a-f]{32})/(?<action>[0-9a-zA-Z]+)/\\?#',
    'RULE' => 'hash=$1&action=$2&',
    'ID' => 'bitrix:disk.external.link',
    'PATH' => '/docs/pub/index.php',
    'SORT' => 100,
  ),

То есть мы не заменяем весь файл, а просто добавляем новый блок первым элементом.

Что делает это правило

Когда открываешь:

/local/mvc/about

Битрикс превращает это в:

/local/mvc/index.php?route=/about

А этот вариант у нас уже работает.

После вставки проверь

Открой:

https://bitrix24-stage.gaz.ru/local/mvc/about

и:

https://bitrix24-stage.gaz.ru/local/mvc/ping

Если всё равно перекинет на карту сайта

Тогда, возможно, PHP держит старую версию файла в OPcache. В Битриксовой PHP-командной строке выполни именно PHP-код:

opcache_reset();
echo 'OPcache reset OK';

Не php -l, а именно этот код.

Потом снова открой:

https://bitrix24-stage.gaz.ru/local/mvc/about

Сейчас главный момент: правило должно быть первым внутри массива $arUrlRewrite.