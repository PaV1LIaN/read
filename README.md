Обработка заняла 6m 3s

API и миграция работают. Ошибка находится во фронтенде редактора.

В Stage 20 функция fillPageForm() вызывает:

setInputValue(...)

но эта функция объявлена только внутри замкнутого модуля 34-editor-ux.js и недоступна остальным JS-файлам. В результате возникает ReferenceError, а общий обработчик показывает «Не удалось загрузить редактор».

Promise fulfilled → undefined в твоём выводе — это нормально.

Быстрая проверка прямо сейчас

В консоли браузера вставь:

window.setInputValue = function (id, value) {
    var element = document.getElementById(id);

    if (element) {
        element.value = value == null ? '' : String(value);
    }
};

(async function () {
    await loadSite();
    await loadPages();
    await loadBlocks();
    await loadAccessList();
    console.log('РЕДАКТОР ЗАГРУЖЕН');
})();

Редактор должен загрузиться без сообщения об ошибке. После обновления страницы временное исправление исчезнет.

Постоянное исправление

Скачай patch:

Скачать исправление загрузки Stage 20

В Termux:

cd ~/sitebuilder

PATCH=$(find ~/storage/downloads \
  -maxdepth 1 \
  -name 'sitebuilder-stage20-editor-load-fix*.patch' \
  | head -1)

echo "$PATCH"
git apply --check "$PATCH"
git apply "$PATCH"

Проверь:

git diff --check
git --no-pager diff --stat

Будут изменены только:

assets/admin/editor/00-core.js
editor.php

В editor.php также увеличена версия подключаемого JS до v=20.1, чтобы браузер не использовал старый файл из кеша.

Создай коммит:

git add assets/admin/editor/00-core.js editor.php

git commit -m "Fix Stage 20 editor initialization"

git push

После обновления этих двух файлов на портале закрой вкладку редактора, открой заново и нажми:

Ctrl + F5

Ещё один найденный дефект данных

У страницы 20 существует секция 6, но блок 28 всё ещё ссылается на старую секцию 17:

block 28 → sectionId 17
существующая секция → ID 6

Это не является причиной текущего падения — редактор умеет временно показать такой блок в первой секции. Но после загрузки редактора выбери блок 28, укажи секцию «Основная секция» и сохрани его размещение. Это устранит повреждённую ссылку в данных.