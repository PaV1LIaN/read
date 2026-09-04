Обработка заняла 1m 6s

Да, причина найдена в header.php: браузер ждёт недоступный StackPath до тайм-аута.

Замените только эту строку:

<script src="https://stackpath.bootstrapcdn.com/bootstrap/4.5.2/js/bootstrap.min.js"></script>

на:

<script src="https://cdn.jsdelivr.net/npm/bootstrap@4.5.2/dist/js/bootstrap.bundle.min.js"></script>

Итоговый блок:

<script src="<?=SITE_TEMPLATE_PATH?>/js/jquery.js"></script>
<script src="https://cdn.jsdelivr.net/npm/bootstrap@4.5.2/dist/js/bootstrap.bundle.min.js"></script>
<script src="<?=SITE_TEMPLATE_PATH?>/js/script.js"></script>

Именно замените старое подключение, не добавляйте второе. Версия bundle нужна для корректной работы модального окна авторизации.

Если jsDelivr также окажется недоступен, файл следует загрузить в папку js активного шаблона и подключить локально:

<script src="<?=SITE_TEMPLATE_PATH?>/js/bootstrap.bundle.min.js"></script>

Шаблон у вас, вероятно, расположен в /bitrix/templates/..., раз /local/templates отсутствует. После сохранения обновите страницу через Ctrl+F5. Ошибка stackpath.bootstrapcdn.com ... ERR_CONNECTION_TIMED_OUT должна исчезнуть.