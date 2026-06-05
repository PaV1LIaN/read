
<!doctype html>
<html lang="ru">
<head>
    <meta charset="UTF-8">

    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
<script data-skip-moving="true">(function() {const canvas = document.createElement('canvas');let gl;try{gl = canvas.getContext('webgl2') || canvas.getContext('webgl') || canvas.getContext('experimental-webgl');}catch (e){return;}if (!gl){return;}const result = {vendor: gl.getParameter(gl.VENDOR),renderer: gl.getParameter(gl.RENDERER),};const debugInfo = gl.getExtension('WEBGL_debug_renderer_info');if (debugInfo){result.unmaskedVendor = gl.getParameter(debugInfo.UNMASKED_VENDOR_WEBGL);result.unmaskedRenderer = gl.getParameter(debugInfo.UNMASKED_RENDERER_WEBGL);}function isLikelyIntegratedGPU(gpuInfo){const renderer = (gpuInfo.unmaskedRenderer || gpuInfo.renderer || '').toLowerCase();const vendor = (gpuInfo.unmaskedVendor || gpuInfo.vendor || '').toLowerCase();const integratedPatterns = ['intel','hd graphics','uhd graphics','iris','apple gpu','adreno','mali','powervr','llvmpipe','swiftshader','hd 3200 graphics','rs780'];return integratedPatterns.some(pattern => renderer.includes(pattern) || vendor.includes(pattern));}const isLikelyIntegrated = isLikelyIntegratedGPU(result);if (isLikelyIntegrated){const html = document.documentElement;html.classList.add('bx-integrated-gpu', '--ui-reset-bg-blur');}})();</script>


<link href="/bitrix/js/intranet/intranet-common.min.css?175621980261199" type="text/css"  rel="stylesheet" />
<link href="/bitrix/js/ui/design-tokens/dist/ui.design-tokens.min.css?175621985523463" type="text/css"  rel="stylesheet" />
<link href="/bitrix/js/intranet/design-tokens/bitrix24/air-design-tokens.min.css?17562210193744" type="text/css"  rel="stylesheet" />
<link href="/bitrix/js/ui/fonts/opensans/ui.font.opensans.min.css?17562198552320" type="text/css"  rel="stylesheet" />
<link href="/bitrix/js/main/popup/dist/main.popup.bundle.min.css?175622046728056" type="text/css"  rel="stylesheet" />
<link href="/bitrix/js/main/loader/dist/loader.bundle.min.css?17562197422029" type="text/css"  rel="stylesheet" />
<link href="/bitrix/js/main/core/css/core_viewer.min.css?175621974158384" type="text/css"  rel="stylesheet" />
<link href="/bitrix/js/ui/icon-set/icon-base.min.css?17562205141604" type="text/css"  rel="stylesheet" />
<link href="/bitrix/js/ui/icon-set/actions/style.min.css?175622051419578" type="text/css"  rel="stylesheet" />
<link href="/bitrix/js/ui/icon-set/main/style.min.css?175622051474857" type="text/css"  rel="stylesheet" />
<link href="/bitrix/js/ui/viewer/dist/viewer.bundle.min.css?175621985525135" type="text/css"  rel="stylesheet" />
<link href="/bitrix/cache/css/s1/bitrix24/kernel_ui_notification/kernel_ui_notification_v1.css?17788291521958" type="text/css"  rel="stylesheet" />
<link href="/bitrix/js/disk/css/disk.min.css?177640680277232" type="text/css"  rel="stylesheet" />
<script>if(!window.BX)window.BX={};if(!window.BX.message)window.BX.message=function(mess){if(typeof mess==='object'){for(let i in mess) {BX.message[i]=mess[i];} return true;}};</script>
<script>(window.BX||top.BX).message({"JS_CORE_LOADING":"Загрузка...","JS_CORE_NO_DATA":"- Нет данных -","JS_CORE_WINDOW_CLOSE":"Закрыть","JS_CORE_WINDOW_EXPAND":"Развернуть","JS_CORE_WINDOW_NARROW":"Свернуть в окно","JS_CORE_WINDOW_SAVE":"Сохранить","JS_CORE_WINDOW_CANCEL":"Отменить","JS_CORE_WINDOW_CONTINUE":"Продолжить","JS_CORE_H":"ч","JS_CORE_M":"м","JS_CORE_S":"с","JSADM_AI_HIDE_EXTRA":"Скрыть лишние","JSADM_AI_ALL_NOTIF":"Показать все","JSADM_AUTH_REQ":"Требуется авторизация!","JS_CORE_WINDOW_AUTH":"Войти","JS_CORE_IMAGE_FULL":"Полный размер"});</script>

<script src="/bitrix/js/main/core/core.min.js?1756220918229643"></script>

<script>BX.Runtime.registerExtension({"name":"main.core","namespace":"BX","loaded":true});</script>
<script>BX.setJSList(["\/bitrix\/js\/main\/core\/core_ajax.js","\/bitrix\/js\/main\/core\/core_promise.js","\/bitrix\/js\/main\/polyfill\/promise\/js\/promise.js","\/bitrix\/js\/main\/loadext\/loadext.js","\/bitrix\/js\/main\/loadext\/extension.js","\/bitrix\/js\/main\/polyfill\/promise\/js\/promise.js","\/bitrix\/js\/main\/polyfill\/find\/js\/find.js","\/bitrix\/js\/main\/polyfill\/includes\/js\/includes.js","\/bitrix\/js\/main\/polyfill\/matches\/js\/matches.js","\/bitrix\/js\/ui\/polyfill\/closest\/js\/closest.js","\/bitrix\/js\/main\/polyfill\/fill\/main.polyfill.fill.js","\/bitrix\/js\/main\/polyfill\/find\/js\/find.js","\/bitrix\/js\/main\/polyfill\/matches\/js\/matches.js","\/bitrix\/js\/main\/polyfill\/core\/dist\/polyfill.bundle.js","\/bitrix\/js\/main\/core\/core.js","\/bitrix\/js\/main\/polyfill\/intersectionobserver\/js\/intersectionobserver.js","\/bitrix\/js\/main\/lazyload\/dist\/lazyload.bundle.js","\/bitrix\/js\/main\/polyfill\/core\/dist\/polyfill.bundle.js","\/bitrix\/js\/main\/parambag\/dist\/parambag.bundle.js"]);
</script>
<script>BX.Runtime.registerExtension({"name":"ls","namespace":"window","loaded":true});</script>
<script>BX.Runtime.registerExtension({"name":"pull.protobuf","namespace":"BX","loaded":true});</script>
<script>BX.Runtime.registerExtension({"name":"rest.client","namespace":"window","loaded":true});</script>
<script>(window.BX||top.BX).message({"pull_server_enabled":"Y","pull_config_timestamp":1757659923,"shared_worker_allowed":"Y","pull_guest_mode":"N","pull_guest_user_id":0,"pull_worker_mtime":1756220331});(window.BX||top.BX).message({"PULL_OLD_REVISION":"Для продолжения корректной работы с сайтом необходимо перезагрузить страницу."});</script>
<script>BX.Runtime.registerExtension({"name":"pull.client","namespace":"BX","loaded":true});</script>
<script>BX.Runtime.registerExtension({"name":"pull","namespace":"window","loaded":true});</script>
<script>BX.Runtime.registerExtension({"name":"intranet.design-tokens.bitrix24","namespace":"window","loaded":true});</script>
<script>BX.Runtime.registerExtension({"name":"ui.design-tokens","namespace":"window","loaded":true});</script>
<script>BX.Runtime.registerExtension({"name":"ui.fonts.opensans","namespace":"window","loaded":true});</script>
<script>BX.Runtime.registerExtension({"name":"main.popup","namespace":"BX.Main","loaded":true});</script>
<script>BX.Runtime.registerExtension({"name":"popup","namespace":"window","loaded":true});</script>
<script>BX.Runtime.registerExtension({"name":"main.loader","namespace":"BX","loaded":true});</script>
<script>BX.Runtime.registerExtension({"name":"loader","namespace":"window","loaded":true});</script>
<script>(window.BX||top.BX).message({"DISK_MYOFFICE":false});(window.BX||top.BX).message({"JS_CORE_VIEWER_DOWNLOAD":"Скачать","JS_CORE_VIEWER_EDIT":"Редактировать","JS_CORE_VIEWER_DESCR_AUTHOR":"Автор","JS_CORE_VIEWER_DESCR_LAST_MODIFY":"Последние изменения","JS_CORE_VIEWER_TOO_BIG_FOR_VIEW":"Файл слишком большой для просмотра","JS_CORE_VIEWER_OPEN_WITH_GVIEWER":"Открыть файл в Google Viewer","JS_CORE_VIEWER_IFRAME_DESCR_ERROR":"К сожалению, не удалось открыть документ.","JS_CORE_VIEWER_IFRAME_PROCESS_SAVE_DOC":"Сохранение документа","JS_CORE_VIEWER_IFRAME_UPLOAD_DOC_TO_GOOGLE":"Загрузка документа","JS_CORE_VIEWER_IFRAME_CONVERT_ACCEPT":"Конвертировать","JS_CORE_VIEWER_IFRAME_CONVERT_DECLINE":"Отменить","JS_CORE_VIEWER_IFRAME_CONVERT_TO_NEW_FORMAT":"Документ будет сконвертирован в docx, xls, pptx, так как имеет старый формат.","JS_CORE_VIEWER_IFRAME_DESCR_SAVE_DOC":"Сохранить документ?","JS_CORE_VIEWER_IFRAME_SAVE_DOC":"Сохранить","JS_CORE_VIEWER_IFRAME_DISCARD_DOC":"Отменить изменения","JS_CORE_VIEWER_IFRAME_CHOICE_SERVICE_EDIT":"Редактировать с помощью","JS_CORE_VIEWER_IFRAME_SET_DEFAULT_SERVICE_EDIT":"Использовать для всех файлов","JS_CORE_VIEWER_IFRAME_CHOICE_SERVICE_EDIT_ACCEPT":"Применить","JS_CORE_VIEWER_IFRAME_CHOICE_SERVICE_EDIT_DECLINE":"Отменить","JS_CORE_VIEWER_IFRAME_UPLOAD_NEW_VERSION_IN_COMMENT":"Загрузил новую версию файла","JS_CORE_VIEWER_SERVICE_GOOGLE_DRIVE":"Google Docs","JS_CORE_VIEWER_SERVICE_SKYDRIVE":"MS Office Online","JS_CORE_VIEWER_IFRAME_CANCEL":"Отмена","JS_CORE_VIEWER_IFRAME_DESCR_SAVE_DOC_F":"В одном из окон вы редактируете данный документ. Если вы завершили работу над документом, нажмите \u0022#SAVE_DOC#\u0022, чтобы загрузить измененный файл на портал.","JS_CORE_VIEWER_SAVE":"Сохранить","JS_CORE_VIEWER_EDIT_IN_SERVICE":"Редактировать в #SERVICE#","JS_CORE_VIEWER_NOW_EDITING_IN_SERVICE":"Редактирование в #SERVICE#","JS_CORE_VIEWER_SAVE_TO_OWN_FILES_MSGVER_1":"Сохранить на Битрикс24.Диск","JS_CORE_VIEWER_DOWNLOAD_TO_PC":"Скачать на локальный компьютер","JS_CORE_VIEWER_GO_TO_FILE":"Перейти к файлу","JS_CORE_VIEWER_DESCR_SAVE_FILE_TO_OWN_FILES":"Файл #NAME# успешно сохранен\u003Cbr\u003Eв папку \u0022Файлы\\Сохраненные\u0022","JS_CORE_VIEWER_DESCR_PROCESS_SAVE_FILE_TO_OWN_FILES":"Файл #NAME# сохраняется\u003Cbr\u003Eна ваш \u0022Битрикс24.Диск\u0022","JS_CORE_VIEWER_HISTORY_ELEMENT":"История","JS_CORE_VIEWER_VIEW_ELEMENT":"Просмотреть","JS_CORE_VIEWER_THROUGH_VERSION":"Версия #NUMBER#","JS_CORE_VIEWER_THROUGH_LAST_VERSION":"Последняя версия","JS_CORE_VIEWER_DISABLE_EDIT_BY_PERM":"Автор не разрешил вам редактировать этот документ","JS_CORE_VIEWER_IFRAME_UPLOAD_NEW_VERSION_IN_COMMENT_F":"Загрузила новую версию файла","JS_CORE_VIEWER_IFRAME_UPLOAD_NEW_VERSION_IN_COMMENT_M":"Загрузил новую версию файла","JS_CORE_VIEWER_IFRAME_CONVERT_TO_NEW_FORMAT_EX":"Документ будет сконвертирован в формат #NEW_FORMAT#, так как текущий формат #OLD_FORMAT# является устаревшим.","JS_CORE_VIEWER_CONVERT_TITLE":"Конвертировать в #NEW_FORMAT#?","JS_CORE_VIEWER_CREATE_IN_SERVICE":"Создать с помощью #SERVICE#","JS_CORE_VIEWER_NOW_CREATING_IN_SERVICE":"Создание документа в #SERVICE#","JS_CORE_VIEWER_SAVE_AS":"Сохранить как","JS_CORE_VIEWER_CREATE_DESCR_SAVE_DOC_F":"В одном из окон вы создаете новый документ. Если вы завершили работу над документом, нажмите \u0022#SAVE_AS_DOC#\u0022, чтобы перейти к добавлению документа на портал.","JS_CORE_VIEWER_NOW_DOWNLOAD_FROM_SERVICE":"Загрузка документа из #SERVICE#","JS_CORE_VIEWER_EDIT_IN_LOCAL_SERVICE":"Редактировать на моём компьютере","JS_CORE_VIEWER_EDIT_IN_LOCAL_SERVICE_SHORT":"Редактировать на #SERVICE#","JS_CORE_VIEWER_SERVICE_LOCAL":"моём компьютере","JS_CORE_VIEWER_DOWNLOAD_B24_DESKTOP":"Скачать","JS_CORE_VIEWER_SERVICE_LOCAL_INSTALL_DESKTOP_MSGVER_1":"Для эффективного редактирования документов на компьютере, установите приложение для компьютера и подключите Битрикс24.Диск","JS_CORE_VIEWER_SHOW_FILE_DIALOG_OAUTH_NOTICE":"Для просмотра файла, пожалуйста, авторизуйтесь в своем аккаунте \u003Ca id=\u0022bx-js-disk-run-oauth-modal\u0022 href=\u0022#\u0022\u003E#SERVICE#\u003C\/a\u003E.","JS_CORE_VIEWER_SERVICE_OFFICE365":"Office365","JS_CORE_VIEWER_DOCUMENT_IS_LOCKED_BY":"Документ заблокирован на редактирование","JS_CORE_VIEWER_SERVICE_MYOFFICE":"МойОфис","JS_CORE_VIEWER_OPEN_PDF_PREVIEW":"Просмотреть pdf-версию файла","JS_CORE_VIEWER_AJAX_ACCESS_DENIED":"Не хватает прав для просмотра файла. Попробуйте обновить страницу.","JS_CORE_VIEWER_AJAX_CONNECTION_FAILED":"При попытке открыть файл возникла ошибка. Пожалуйста, попробуйте позже.","JS_CORE_VIEWER_AJAX_OPEN_NEW_TAB":"Открыть в новом окне","JS_CORE_VIEWER_AJAX_PRINT":"Распечатать","JS_CORE_VIEWER_TRANSFORMATION_IN_PROCESS":"Документ сохранён. Мы готовим его к показу.","JS_CORE_VIEWER_IFRAME_ERROR_TITLE":"Не удалось открыть документ","JS_CORE_VIEWER_DOWNLOAD_B24_DESKTOP_FULL":"Скачать приложение","JS_CORE_VIEWER_DOWNLOAD_DOCUMENT":"Скачать документ","JS_CORE_VIEWER_IFRAME_ERROR_COULD_NOT_VIEW":"К сожалению, не удалось просмотреть документ.","JS_CORE_VIEWER_ACTIONPANEL_MORE":"Ещё"});</script>
<script>BX.Runtime.registerExtension({"name":"viewer","namespace":"window","loaded":true});</script>
<script>BX.Runtime.registerExtension({"name":"ui.icon-set","namespace":"window","loaded":true});</script>
<script>BX.Runtime.registerExtension({"name":"ui.icon-set.actions","namespace":"window","loaded":true});</script>
<script>BX.Runtime.registerExtension({"name":"ui.icon-set.main","namespace":"window","loaded":true});</script>
<script>BX.Runtime.registerExtension({"name":"ui.icons.generator","namespace":"BX.UI.Icons.Generator","loaded":true});</script>
<script>(window.BX||top.BX).message({"JS_UI_VIEWER_DEFAULT_ERROR_TITLE":"Произошла ошибка","JS_UI_VIEWER_IMAGE_VIEW_FULL_SIZE_MSGVER_1":"Открыть оригинал","JS_UI_VIEWER_ITEM_ACTION_DOWNLOAD":"Скачать","JS_UI_VIEWER_ITEM_ACTION_EDIT":"Редактировать","JS_UI_VIEWER_ITEM_ACTION_SHARE":"Поделиться","JS_UI_VIEWER_ITEM_ACTION_DELETE":"Удалить","JS_UI_VIEWER_ITEM_UNKNOWN_TITLE":"Формат файла не поддерживается","JS_UI_VIEWER_ITEM_UNKNOWN_NOTICE":"Но вы можете просмотреть его на компьютере","JS_UI_VIEWER_ITEM_UNKNOWN_DOWNLOAD_ACTION":"Скачать файл","JS_UI_VIEWER_ITEM_TRANSFORMATION_ERROR":"Ошибка при конвертации. Не удалось открыть файл","JS_UI_VIEWER_ITEM_TRANSFORMATION_IN_PROGRESS":"Идет конвертация","JS_UI_VIEWER_ITEM_TRANSFORMATION_TIMEOUT":"Неудачная конвертация. Вы можете скачать файл","JS_UI_VIEWER_ITEM_TRANSFORMATION_ERROR_1":"Не удалось открыть файл. Вы можете его \u003Ca href=\u0022#DOWNLOAD_LINK#\u0022 target=\u0022_blank\u0022\u003Eскачать\u003C\/a\u003E.","JS_UI_VIEWER_ITEM_TRANSFORMATION_HINT":"\u003Ca href=\u0022#\u0022 onclick=\u0027top.BX.Helper.show(\u0022redirect=detail\u0026code=8775937\u0022);event.preventDefault();\u0027\u003EПодробнее\u003C\/a\u003E о возможных причинах.","JS_UI_VIEWER_ITEM_PREPARING_TO_PRINT":"Подготовка документа к печати: #PROGRESS#%","JS_UI_VIEWER_SINGLE_DOCUMENT_LISTING_PAGES":"Стр. #CURRENT#\u003Cdiv class=\u0022ui-viewer__single-document--listing-pages-all\u0022\u003E\/#ALL#\u003C\/div\u003E","JS_UI_VIEWER_SINGLE_DOCUMENT_SCALE_ZOOM_IN":"Увеличить","JS_UI_VIEWER_SINGLE_DOCUMENT_SCALE_ZOOM_OUT":"Уменьшить","JS_UI_VIEWER_ITEM_ACTION_INFO":"Подробнее","JS_UI_VIEWER_ITEM_ACTION_PRINT":"Печать","JS_UI_VIEWER_ITEM_ACTION_ROTATE":"Перевернуть"});</script>
<script>BX.Runtime.registerExtension({"name":"ui.viewer","namespace":"BX.UI.Viewer","loaded":true});</script>
<script>BX.Runtime.registerExtension({"name":"fx","namespace":"window","loaded":true});</script>
<script>BX.Runtime.registerExtension({"name":"dd","namespace":"window","loaded":true});</script>
<script>BX.Runtime.registerExtension({"name":"ui.notification","namespace":"window","loaded":true});</script>
<script>(window.BX||top.BX).message({"disk_restriction":false,"disk_onlyoffice_available":false,"disk_revision_api":8,"disk_document_service":"gdrive"});(window.BX||top.BX).message({"DISK_JS_STATUS_ACTION_SUCCESS":"Успешно","DISK_JS_STATUS_ACTION_ERROR":"Произошла ошибка","DISK_JS_SHARING_LABEL_RIGHTS_FOLDER":"Общий доступ к папке","DISK_JS_SHARING_LABEL_NAME_RIGHTS_USER":"Пользователи","DISK_JS_SHARING_LABEL_NAME_RIGHTS":"Права доступа","DISK_JS_SHARING_LABEL_NAME_ADD_RIGHTS_USER":"Добавить ещё","DISK_JS_SHARING_LABEL_NAME_ALLOW_SHARING_RIGHTS_USER":"Разрешить настройку общего доступа","DISK_JS_SHARING_LABEL_RIGHT_READ":"Чтение","DISK_JS_SHARING_LABEL_RIGHT_EDIT":"Редактирование","DISK_JS_SHARING_LABEL_RIGHT_FULL":"Полный доступ","DISK_JS_SHARING_LABEL_TOOLTIP_SHARING":"Все сотрудники, имеющие доступ к папке,\u003Cbr\/\u003Eсмогут добавлять новых участников общего доступа","DISK_JS_SHARING_LABEL_OWNER":"Владелец","DISK_JS_SHARING_LABEL_TITLE_MODAL":"Параметры общего доступа","DISK_JS_BTN_CANCEL":"Отменить","DISK_JS_BTN_CLOSE":"Закрыть","DISK_JS_BTN_SAVE":"Сохранить","DISK_JS_BTN_DOWNLOAD":"Скачать","DISK_JS_SHARING_LABEL_TITLE_MODAL_2":"Общий доступ","DISK_JS_SHARING_LABEL_TITLE_MODAL_3":"Общий доступ","DISK_JS_SERVICE_CHOICE_TITLE_SMALL":"Выбор способа работы","DISK_JS_SERVICE_CHOICE_TITLE":"Выберите удобный способ работы с документами","DISK_JS_SERVICE_CHANGE_TEXT":"Изменить выбор можно будет по ссылке Еще или в настройках Диска","DISK_JS_SERVICE_LOCAL_TITLE":"Локально","DISK_JS_SERVICE_LOCAL_TEXT":"Документы всегда будут открываться в программе на вашем компьютере","DISK_JS_SERVICE_CLOUD_TITLE":"В облаке","DISK_JS_SERVICE_CLOUD_TEXT":"Вы сможете работать с документами через Google Docs или MS Office Web App","DISK_JS_SERVICE_HELP_TEXT":"Изменить ваш выбор можно будет по ссылке Еще","DISK_JS_SERVICE_HELP_TEXT_2":"Изменить ваш выбор можно будет в настройках","DISK_JS_SHARING_LABEL_RIGHT_ADD":"Добавление","DISK_JS_USER_LOCKED_DOCUMENT":"Заблокировал документ","DISK_JS_PLAYER_ERROR_MESSAGE":"К сожалению, ваш браузер не может воспроизвести этот файл.\u003Cbr \/\u003EВы можете \u003Cspan class=\u0022disk-player-download\u0022\u003Eскачать\u003C\/span\u003E и посмотреть его на компьютере","DISK_VIEWER_DESCR_SAVE_FILE_TO_OWN_FILES":"Файл #NAME# успешно сохранен\u003Cbr\u003Eв папку \u0022Файлы\\Сохраненные\u0022","DISK_VIEWER_DESCR_PROCESS_SAVE_FILE_TO_OWN_FILES":"Файл #NAME# сохраняется\u003Cbr\u003Eна ваш \u0022Битрикс24.Диск\u0022","DISK_JS_HELP_WITH_BDISK":"Подробнее о работе с документами","DISK_JS_SERVICE_B24_DOCS_TITLE":"Битрикс24","DISK_JS_SERVICE_B24_DOCS_TEXT":"Вы сможете работать с ними совместно. Редактировать, обсуждать, делиться.","DISK_JS_DOCUMENT_ONLYOFFICE_SAVE_PROCESS":"Идёт сохранение документа: #name#","DISK_JS_DOCUMENT_ONLYOFFICE_SAVED":"Документ #name# обновлён"});</script>
<script>BX.Runtime.registerExtension({"name":"disk","namespace":"window","loaded":true});</script>
<script>BX.Runtime.registerExtension({"name":"disk.viewer.onlyoffice-item","namespace":"BX.Disk.Viewer","loaded":true});</script>
<script>(window.BX||top.BX).message({"JS_VIEWER_DOCUMENT_ITEM_SHOW_FILE_DIALOG_OAUTH_NOTICE":"Для просмотра файла, пожалуйста, авторизуйтесь в своем аккаунте \u003Ca id=\u0022bx-js-disk-run-oauth-modal\u0022 href=\u0022#\u0022\u003E#SERVICE#\u003C\/a\u003E.","JS_VIEWER_DOCUMENT_ITEM_OPEN_FILE_OFFICE365":"Открыть","JS_VIEWER_DOCUMENT_ITEM_OPEN_DESCR_OFFICE365":"Для просмотра файла необходимо его открыть в новом окне","JS_VIEWER_DOCUMENT_ITEM_OPEN_HELP_HINT_OFFICE365":"\u003Ca href=\u0022#\u0022 onclick=\u0027top.BX.Helper.show(\u0022redirect=detail\u0026code=9562101\u0022);event.preventDefault();\u0027\u003EПодробнее\u003C\/a\u003E о работе с файлами через Office365.","JS_VIEWER_DOCUMENT_ONLYOFFICE_SAVE_PROCESS":"Идёт сохранение документа: #name#","JS_VIEWER_DOCUMENT_ONLYOFFICE_SAVED":"Документ #name# обновлён"});</script>
<script>BX.Runtime.registerExtension({"name":"disk.viewer.document-item","namespace":"window","loaded":true});</script>
<script>(window.BX||top.BX).message({"LANGUAGE_ID":"ru","FORMAT_DATE":"DD.MM.YYYY","FORMAT_DATETIME":"DD.MM.YYYY HH:MI:SS","COOKIE_PREFIX":"BITRIX_SM","SERVER_TZ_OFFSET":"10800","UTF_MODE":"Y","SITE_ID":"s1","SITE_DIR":"\/","USER_ID":"1","SERVER_TIME":1780638308,"USER_TZ_OFFSET":0,"USER_TZ_AUTO":"Y","bitrix_sessid":"b658aa53af5cd4bb145323d21d092687"});</script>


<script  src="/bitrix/cache/js/s1/bitrix24/kernel_main/kernel_main_v1.js?1778829674208977"></script>
<script src="/bitrix/js/main/core/core_ls.min.js?17562197412683"></script>
<script src="/bitrix/js/pull/protobuf/protobuf.min.js?175621981976433"></script>
<script src="/bitrix/js/pull/protobuf/model.min.js?175621981914190"></script>
<script src="/bitrix/js/rest/client/rest.client.min.js?17562198229240"></script>
<script src="/bitrix/js/pull/client/pull.client.min.js?175622033449849"></script>
<script src="/bitrix/js/main/popup/dist/main.popup.bundle.min.js?175622091866986"></script>
<script src="/bitrix/js/main/loader/dist/loader.bundle.min.js?17562197424392"></script>
<script src="/bitrix/js/main/core/core_viewer.min.js?175622091899239"></script>
<script src="/bitrix/js/ui/icons/generator/dist/ui.icons.generator.bundle.min.js?175621985410114"></script>
<script src="/bitrix/js/ui/viewer/ui.viewer.item.min.js?175622051421144"></script>
<script src="/bitrix/js/ui/viewer/ui.viewer.min.js?175622050734566"></script>
<script src="/bitrix/js/ui/viewer/dist/viewer.bundle.min.js?175621985519611"></script>
<script  src="/bitrix/cache/js/s1/bitrix24/kernel_ui_notification/kernel_ui_notification_v1.js?177882915217837"></script>
<script src="/bitrix/js/disk/c_disk.min.js?177640680223189"></script>
<script src="/bitrix/js/disk/viewer/onlyoffice-item/dist/disk.onlyoffice-item.bundle.min.js?17764068022981"></script>
<script src="/bitrix/js/disk/viewer/document-item/item.min.js?17764068024148"></script>
<script>BX.setJSList(["\/bitrix\/js\/main\/session.js","\/bitrix\/js\/main\/pageobject\/dist\/pageobject.bundle.js","\/bitrix\/js\/main\/core\/core_window.js","\/bitrix\/js\/main\/core\/core_fx.js","\/bitrix\/js\/main\/date\/main.date.js","\/bitrix\/js\/main\/core\/core_dd.js","\/bitrix\/js\/main\/dd.js","\/bitrix\/js\/main\/core\/core_uf.js","\/bitrix\/js\/main\/core\/core_date.js","\/bitrix\/js\/main\/core\/core_timer.js","\/bitrix\/js\/main\/utils.js","\/bitrix\/js\/main\/rating_like.js","\/bitrix\/js\/main\/core\/core_tooltip.js","\/bitrix\/js\/main\/core\/core_autosave.js","\/bitrix\/js\/ui\/notification\/ui.notification.balloon.js","\/bitrix\/js\/ui\/notification\/ui.notification.stack.js","\/bitrix\/js\/ui\/notification\/ui.notification.center.js"]);</script>
<script>BX.setCSSList(["\/bitrix\/js\/ui\/notification\/ui.notification.css"]);</script>
<script>
BX.message({"SessExpired": 'Ваш сеанс работы с сайтом завершен из-за отсутствия активности в течение 15 мин. Введенные на странице данные не будут сохранены. Скопируйте их перед тем, как закроете или обновите страницу.'});
bxSession.Expand('b658aa53af5cd4bb145323d21d092687.e28867c61538e2f6cf222b45898104495b6b467e822e0c2d68263d31920509c3');
</script>
<script>
					if (Intl && Intl.DateTimeFormat)
					{
						const timezone = Intl.DateTimeFormat().resolvedOptions().timeZone;
						document.cookie = "BITRIX_SM_TZ=" + timezone + "; path=/; expires=Tue, 01 Jun 2027 00:00:00 +0300";
						
						if (timezone !== "Europe/Moscow")
						{
							BX.ready(function () {
								BX.ajax.runAction(
									"main.timezone.set",
									{
										data: {
											timezone: timezone
										}
									}
								);
							});
						}
				
					}
				</script>




    <title>Диск</title>

    <link rel="stylesheet" href="/local/sitebuilder/assets/public/public.css?v=9">

            <link rel="stylesheet" href="/local/sitebuilder/components/disk/styles.css?v=4">
    
    <style>
        :root {
            --sb-accent: #2563eb;
            --sb-container-width: 1920px;
            --sb-left-width: 260px;
            --sb-right-width: 260px;
        }
    </style>
</head>
<body>
<div class="sb-public-shell" style="--sb-accent: #2563eb; --sb-logo-size: 60px; background-color: #ffffff; background-image: url(&quot;/upload/sitebuilder/appearance/d5c/lbunf7c3uzxf3sjln0xszxgkoann28lv/logo_gaz.png&quot;); background-size: auto; background-position: center center; background-repeat: repeat">
            <header class="sb-public-header">
            <div class="sb-container sb-header-container">
                <div class="sb-header-brand-row">
                    <div class="sb-brand">
                        <span class="sb-brand__logo"><img src="/upload/sitebuilder/appearance/00d/tn0g0wdasxohu2bvlzgcw64xgjxs3drg/company.png" alt="Тестовый сайт"></span><span class="sb-brand__text">Тестовый сайт</span>                    </div>

                                    </div>

                                    <div class="sb-header-menu-row">
                        <nav class="sb-public-menu"><div class="sb-public-menu__item is-active"><a class="sb-public-menu__link" href="/local/sitebuilder/public.php?siteId=13&amp;pageId=14">Диск</a></div></nav>                    </div>
                            </div>
        </header>
    
    <main class="sb-public-main">
        <div class="sb-container">
            

            <div class="sb-layout sb-layout--left ">
                                    <aside class="sb-sidebar sb-sidebar--left">
                        <div class="sb-box">
                            <div class="sb-section-nav"><div class="sb-section-nav__title-row">  <a class="sb-section-nav__root-link" href="/local/sitebuilder/public.php?siteId=13&amp;pageId=14">Диск</a></div><div class="sb-section-nav__tree"><div class="sb-tree-node" style="--sb-nav-depth:0;">  <div class="sb-tree-node__row">    <span class="sb-tree-node__toggle sb-tree-node__toggle--empty"></span>    <a class="sb-section-nav__link" href="/local/sitebuilder/public.php?siteId=13&amp;pageId=18">      <span class="sb-section-nav__text">Диск 2</span>    </a>  </div></div></div></div>                        </div>
                    </aside>
                
                <section class="sb-content">
                    <div class="sb-box sb-box--content">
                                                    <h1 class="sb-page-title">
                                Диск                            </h1>

                            
                            <div class="sb-page-sections"><section class="sb-page-section sb-page-section--columns-1"><div class="sb-page-section__grid" style="display:grid;grid-template-columns:repeat(1,minmax(0,1fr));gap:24px;width:100%;min-width:0;align-items:start;box-sizing:border-box;"><div class="sb-page-section__column sb-page-section__column--1" style="min-width:0;box-sizing:border-box;"><div class="sb-disk"
     id="sb-disk-17"
     data-site-id="13"
     data-page-id="14"
     data-block-id="17"
     data-sessid="b658aa53af5cd4bb145323d21d092687"
     data-initial-state="{&quot;siteId&quot;:13,&quot;pageId&quot;:14,&quot;blockId&quot;:17,&quot;rootFolderId&quot;:408,&quot;rootSource&quot;:&quot;site&quot;,&quot;currentFolderId&quot;:408,&quot;settings&quot;:{&quot;title&quot;:&quot;Файлы&quot;,&quot;rootMode&quot;:&quot;site&quot;,&quot;rootFolderId&quot;:null,&quot;viewMode&quot;:&quot;table&quot;,&quot;allowUpload&quot;:true,&quot;allowCreateFolder&quot;:true,&quot;allowRename&quot;:true,&quot;allowDelete&quot;:true,&quot;allowDownload&quot;:true,&quot;showSearch&quot;:true,&quot;showBreadcrumbs&quot;:true,&quot;defaultSort&quot;:&quot;updatedAt&quot;,&quot;defaultSortDirection&quot;:&quot;desc&quot;,&quot;allowedExtensions&quot;:[],&quot;maxFileSize&quot;:52428800,&quot;permissionMode&quot;:&quot;inherit_site&quot;,&quot;useSiteRootFallback&quot;:true},&quot;permissions&quot;:{&quot;canView&quot;:true,&quot;canUpload&quot;:true,&quot;canCreateFolder&quot;:true,&quot;canRename&quot;:true,&quot;canDelete&quot;:true,&quot;canDownload&quot;:true,&quot;canManageAccess&quot;:true,&quot;canEditSettings&quot;:true,&quot;role&quot;:&quot;bitrix_admin&quot;}}">

    <div class="sb-disk__header">
        <div class="sb-disk__header-main">
            <div class="sb-disk__title-wrap">
                <h3 class="sb-disk__title">Файлы</h3>
                <div class="sb-disk__subtitle" data-role="subtitle"></div>
            </div>
        </div>

        <div class="sb-disk__header-actions">
            <button type="button" class="sb-disk__btn sb-disk__btn--ghost" data-action="refresh">Обновить</button>

                            <button type="button" class="sb-disk__btn sb-disk__btn--ghost" data-action="settings">Настройки</button>
                    </div>
    </div>

            <div class="sb-disk__breadcrumbs" data-role="breadcrumbs"></div>
    
    <div class="sb-disk__toolbar">
        <div class="sb-disk__toolbar-left">
                            <div class="sb-disk__search">
                    <input type="text"
                           class="sb-disk__search-input"
                           data-role="search-input"
                           placeholder="Поиск файлов и папок">
                </div>
            
            <select class="sb-disk__select" data-role="sort-select">
                <option value="updatedAt:desc">Сначала новые</option>
                <option value="updatedAt:asc">Сначала старые</option>
                <option value="name:asc">По имени А–Я</option>
                <option value="name:desc">По имени Я–А</option>
                <option value="size:desc">По размеру</option>
            </select>
        </div>

        <div class="sb-disk__toolbar-right">
                            <button type="button" class="sb-disk__btn" data-action="upload">Загрузить</button>
            
                            <button type="button" class="sb-disk__btn" data-action="create-folder">Новая папка</button>
            
            <div class="sb-disk__view-switch">
                <button type="button" class="sb-disk__view-btn is-active" data-view="table">Таблица</button>
                <button type="button" class="sb-disk__view-btn" data-view="grid">Плитка</button>
            </div>
        </div>
    </div>

    <div class="sb-disk__bulkbar" data-role="bulkbar" hidden>
        <span class="sb-disk__bulkbar-text" data-role="bulkbar-text">Выбрано: 0</span>
        <div class="sb-disk__bulkbar-actions">
            <button type="button" class="sb-disk__btn" data-action="download-selected">Скачать</button>
            <button type="button" class="sb-disk__btn sb-disk__btn--danger" data-action="delete-selected">Удалить</button>
        </div>
    </div>

    <div class="sb-disk__content">
        <div class="sb-disk__state" data-state="loading" hidden>Загрузка...</div>
        <div class="sb-disk__state" data-state="empty" hidden>Здесь пока нет файлов и папок.</div>
        <div class="sb-disk__state" data-state="error" hidden>Не удалось загрузить содержимое.</div>
        <div class="sb-disk__state" data-state="no-access" hidden>У вас нет доступа к этому разделу.</div>
        <div class="sb-disk__state" data-state="no-root" hidden>
            Для блока не настроена корневая папка.
                            <div class="sb-disk__state-actions">
                    <button type="button" class="sb-disk__btn" data-action="init-site-root">Создать корень сайта</button>
                    <button type="button" class="sb-disk__btn" data-action="init-block-root">Создать папку блока</button>
                </div>
                    </div>

        <div class="sb-disk__view sb-disk__view--table" data-view-container="table">
            <table class="sb-disk__table">
                <thead>
                    <tr>
                        <th class="sb-disk__col sb-disk__col--checkbox">
                            <input type="checkbox" data-role="select-all">
                        </th>
                        <th class="sb-disk__col sb-disk__col--name">Название</th>
                        <th class="sb-disk__col">Тип</th>
                        <th class="sb-disk__col">Размер</th>
                        <th class="sb-disk__col">Изменен</th>
                        <th class="sb-disk__col sb-disk__col--actions"></th>
                    </tr>
                </thead>
                <tbody data-role="items-table"></tbody>
            </table>
        </div>

        <div class="sb-disk__view sb-disk__view--grid" data-view-container="grid" hidden></div>
    </div>

    <input type="file" class="sb-disk__file-input" data-role="upload-input" multiple hidden>

            <div class="sb-disk-modal" data-role="settings-modal" hidden>
            <div class="sb-disk-modal__backdrop" data-action="close-settings"></div>
            <div class="sb-disk-modal__dialog">
                <div class="sb-disk-modal__header">
                    <h3 class="sb-disk-modal__title">Настройки блока “Диск”</h3>
                    <button type="button" class="sb-disk-modal__close" data-action="close-settings">×</button>
                </div>

                <div class="sb-disk-modal__body">
                    <form class="sb-disk-form" data-role="settings-form">
                        <div class="sb-disk-form__grid">
                            <div class="sb-disk-form__field">
                                <label class="sb-disk-form__label">Заголовок блока</label>
                                <input type="text" class="sb-disk-form__input" name="title">
                            </div>

                            <div class="sb-disk-form__field">
                                <label class="sb-disk-form__label">Источник корня</label>
                                <select class="sb-disk-form__select" name="rootFolderId" data-role="root-select">
                                    <option value="">Использовать корень сайта</option>
                                </select>
                            </div>

                            <div class="sb-disk-form__field">
                                <label class="sb-disk-form__label">Вид по умолчанию</label>
                                <select class="sb-disk-form__select" name="viewMode">
                                    <option value="table">Таблица</option>
                                    <option value="grid">Плитка</option>
                                </select>
                            </div>

                            <div class="sb-disk-form__field">
                                <label class="sb-disk-form__label">Сортировка по умолчанию</label>
                                <select class="sb-disk-form__select" name="defaultSort">
                                    <option value="updatedAt">Дата изменения</option>
                                    <option value="createdAt">Дата создания</option>
                                    <option value="name">Имя</option>
                                    <option value="size">Размер</option>
                                </select>
                            </div>

                            <div class="sb-disk-form__field">
                                <label class="sb-disk-form__label">Направление сортировки</label>
                                <select class="sb-disk-form__select" name="defaultSortDirection">
                                    <option value="desc">По убыванию</option>
                                    <option value="asc">По возрастанию</option>
                                </select>
                            </div>

                            <div class="sb-disk-form__field">
                                <label class="sb-disk-form__label">Максимальный размер файла (байт)</label>
                                <input type="number" class="sb-disk-form__input" name="maxFileSize" min="0">
                            </div>

                            <div class="sb-disk-form__field sb-disk-form__field--full">
                                <label class="sb-disk-form__label">Допустимые расширения</label>
                                <input type="text" class="sb-disk-form__input" name="allowedExtensions" placeholder="pdf doc docx xlsx png jpg">
                                <div class="sb-disk-form__hint">Через пробел, запятую или точку с запятой</div>
                            </div>

                            <div class="sb-disk-form__field">
                                <label class="sb-disk-form__label">Режим прав</label>
                                <select class="sb-disk-form__select" name="permissionMode">
                                    <option value="inherit_site">Наследовать права сайта</option>
                                    <option value="custom">Собственные ограничения блока</option>
                                </select>
                            </div>

                            <div class="sb-disk-form__field">
                                <label class="sb-disk-form__check">
                                    <input type="checkbox" name="useSiteRootFallback" value="1">
                                    <span>Использовать корень сайта, если у блока нет своей папки</span>
                                </label>
                            </div>
                        </div>

                        <div class="sb-disk-form__checks">
                            <label class="sb-disk-form__check"><input type="checkbox" name="allowUpload" value="1"><span>Разрешить загрузку</span></label>
                            <label class="sb-disk-form__check"><input type="checkbox" name="allowCreateFolder" value="1"><span>Разрешить создание папок</span></label>
                            <label class="sb-disk-form__check"><input type="checkbox" name="allowRename" value="1"><span>Разрешить переименование</span></label>
                            <label class="sb-disk-form__check"><input type="checkbox" name="allowDelete" value="1"><span>Разрешить удаление</span></label>
                            <label class="sb-disk-form__check"><input type="checkbox" name="allowDownload" value="1"><span>Разрешить скачивание</span></label>
                            <label class="sb-disk-form__check"><input type="checkbox" name="showSearch" value="1"><span>Показывать поиск</span></label>
                            <label class="sb-disk-form__check"><input type="checkbox" name="showBreadcrumbs" value="1"><span>Показывать breadcrumbs</span></label>
                        </div>
                    </form>

                    <div class="sb-disk-modal__message" data-role="settings-message"></div>
                </div>

                <div class="sb-disk-modal__footer">
                    <button type="button" class="sb-disk__btn sb-disk__btn--ghost" data-action="close-settings">Отмена</button>
                    <button type="button" class="sb-disk__btn" data-action="save-settings">Сохранить</button>
                </div>
            </div>
        </div>
    </div>
<div class="sb-public-block sb-public-block--text">
    <div class="sb-public-text">
        sdfsdf    </div>
</div><section class="sb-block sb-block--html">
    <div class="sb-block__inner">
        <div>HTML блок</div>    </div>
</section></div></div></section></div>                                            </div>
                </section>

                            </div>
        </div>
    </main>

    </div>

<script>
document.addEventListener('click', function (e) {
    var toggle = e.target.closest('[data-role="toggle"]');
    if (!toggle) {
        return;
    }

    var node = toggle.closest('.sb-tree-node');
    if (!node) {
        return;
    }

    var isOpen = node.classList.contains('is-open');
    node.classList.toggle('is-open', !isOpen);
    toggle.setAttribute('aria-expanded', !isOpen ? 'true' : 'false');
});
</script>

    <script src="/local/sitebuilder/components/disk/script.js?v=4"></script>

</body>
</html>
