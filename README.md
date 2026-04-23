Нашли точную причину.

У тебя:

data-sessid в HTML есть

BX.bitrix_sessid() существует, но возвращает ''


А текущая логика в getSessid() сначала берет BX.bitrix_sessid(), и не проверяет, что он пустой, поэтому до data-sessid уже не доходит.


---

Что исправить

Файл

/local/sitebuilder/components/disk/script.js

Найди функцию getSessid() и замени строго на это:

DiskComponent.prototype.getSessid = function () {
  if (window.BX && typeof BX.bitrix_sessid === 'function') {
    var bxSessid = BX.bitrix_sessid();
    if (bxSessid) {
      return String(bxSessid);
    }
  }

  var sessidFromData = this.root.getAttribute('data-sessid');
  if (sessidFromData) {
    return String(sessidFromData);
  }

  return '';
};


---

Почему твой текущий вариант не сработал

Сейчас у тебя, скорее всего, так:

if (window.BX && typeof BX.bitrix_sessid === 'function') {
  return BX.bitrix_sessid();
}

И поскольку функция существует, код сразу возвращает '', не доходя до fallback через data-sessid.


---

После правки

Сделай:

1. сохранить script.js


2. жестко обновить страницу
Ctrl + F5


3. снова проверить в консоли:



document.querySelector('.sb-disk')?.getAttribute('data-sessid')

и потом открыть bootstrap в Network


---

Что должно быть дальше

После этой правки EMPTY_SESSID должен уйти.

И тогда либо:

bootstrap вернет ok: true

либо покажет уже следующую реальную проблему:

права

root папка

block context

Disk storage




---

Если хочешь быстро проверить без перезагрузки файла

Можно прямо в консоли временно проверить, что fallback рабочий:

(function () {
  var root = document.querySelector('.sb-disk');
  var bx = (typeof BX !== 'undefined' && typeof BX.bitrix_sessid === 'function') ? BX.bitrix_sessid() : '';
  var data = root ? root.getAttribute('data-sessid') : '';
  console.log('BX:', bx);
  console.log('DATA:', data);
  console.log('RESULT:', bx || data || '');
})();

У тебя RESULT должен быть:

603a4cda044cd6f36550146cc877a7c4

Если после этого вылезет следующая ошибка, пришли bootstrap response.