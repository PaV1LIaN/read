Значит data-viewer на docx отрисовывается, но viewer не стартует на этом элементе. Такое часто бывает со span, динамической перерисовкой и bind.

Надежнее сделать не “магический” bind, а явно открывать viewer по клику.

Что поменять в script.js

Файл:

/local/sitebuilder/components/disk/script.js

1. Добавь в класс метод открытия office viewer

Вставь в файл, например после bindViewer():

DiskComponent.prototype.openOfficeViewer = function (row) {
  if (!row) return false;
  if (!window.BX || !BX.Disk || !BX.Disk.Viewer || !BX.Disk.Viewer.Actions) {
    return false;
  }

  var objectId = Number(row.getAttribute('data-id') || 0);
  var name = row.getAttribute('data-name') || '';
  var src = row.getAttribute('data-preview-url') || '';

  if (objectId <= 0 || !src) {
    return false;
  }

  try {
    var item = new BX.Disk.Viewer.DocumentItem({
      objectId: String(objectId),
      name: name,
      src: src
    });

    BX.Disk.Viewer.Actions.show(item);
    return true;
  } catch (e) {
    console.warn('Office viewer open failed', e);
    return false;
  }
};


---

2. В обработчике open замени блок для файла

Найди в bindStaticEvents() кусок:

} else if (entityType === 'file') {
  var previewMode = row.getAttribute('data-preview-mode') || '';
  var previewUrl = row.getAttribute('data-preview-url') || '';
  var downloadUrl = row.getAttribute('data-download-url') || '';

  if (previewMode === 'office') {
    return;
  }

  if (previewUrl) {
    window.open(previewUrl, '_blank');
  } else if (downloadUrl) {
    window.open(downloadUrl, '_blank');
  }
}

И замени на:

} else if (entityType === 'file') {
  var previewMode = row.getAttribute('data-preview-mode') || '';
  var previewUrl = row.getAttribute('data-preview-url') || '';
  var downloadUrl = row.getAttribute('data-download-url') || '';

  if (previewMode === 'office') {
    if (!self.openOfficeViewer(row) && previewUrl) {
      window.open(previewUrl, '_blank');
    }
    return;
  }

  if (previewUrl) {
    window.open(previewUrl, '_blank');
  } else if (downloadUrl) {
    window.open(downloadUrl, '_blank');
  }
}


---

3. Упрости кнопки Открыть для office-файлов

В renderItemsTable() и renderItemsGrid() не нужно больше генерировать span с data-viewer.
Для всех файлов оставь обычную кнопку:

В renderItemsTable() внутри actions:

openControl = '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>';

В renderItemsGrid() тоже:

openControl = '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>';

То есть убери ветку с:

item.previewMode === 'office'

и data-viewer.


---

Почему это должно помочь

Сейчас у тебя:

картинки открываются, потому что идут через обычный window.open(previewUrl)

docx не открывается, потому что viewer не цепляется к динамически созданному span


После правки:

кнопка Открыть сама вызывает BX.Disk.Viewer.Actions.show(...)

если viewer не поднимется, будет fallback на previewUrl



---

После этого

1. сохрани script.js


2. Ctrl + F5


3. снова открой docx



Если после этого все равно ничего не произойдет, пришли ошибку из консоли после клика по docx.