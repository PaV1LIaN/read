Это уже не maxFileSize компонента.
Раз прогресс дошёл до 100%, браузер файл отправил. Ошибка:

UPLOAD_BAD_RESPONSE

значит upload вернул не JSON, а HTML/текст ошибки. Сейчас твой apiUploadWithProgress просто скрывает настоящую причину.

Сделай одну правку.


---

Замени apiUploadWithProgress в script.js

Файл:

/local/sitebuilder/components/disk/script.js

Найди:

DiskComponent.prototype.apiUploadWithProgress = function (action, formData, onProgress) {

Замени весь метод целиком на этот:

DiskComponent.prototype.apiUploadWithProgress = function (action, formData, onProgress) {
  return new Promise(function (resolve, reject) {
    var xhr = new XMLHttpRequest();

    xhr.open(
      'POST',
      '/local/sitebuilder/components/disk/api.php?action=' + encodeURIComponent(action),
      true
    );

    xhr.upload.addEventListener('progress', function (event) {
      if (!event.lengthComputable) {
        return;
      }

      if (typeof onProgress === 'function') {
        onProgress({
          loaded: event.loaded,
          total: event.total,
          percent: event.total > 0 ? Math.round((event.loaded / event.total) * 100) : 0
        });
      }
    });

    xhr.addEventListener('load', function () {
      var text = xhr.responseText || '';
      var json = null;

      try {
        json = JSON.parse(text);
      } catch (e) {
        console.error('Upload API returned non JSON. HTTP status:', xhr.status);
        console.error('Upload API raw response:', text);

        var cleanText = String(text || '')
          .replace(/<script[\s\S]*?<\/script>/gi, ' ')
          .replace(/<style[\s\S]*?<\/style>/gi, ' ')
          .replace(/<[^>]*>/g, ' ')
          .replace(/\s+/g, ' ')
          .trim()
          .slice(0, 700);

        if (!cleanText) {
          cleanText = 'Пустой ответ сервера';
        }

        reject(new Error(
          'UPLOAD_BAD_RESPONSE. HTTP ' + xhr.status + '. Ответ сервера: ' + cleanText
        ));

        return;
      }

      resolve(json);
    });

    xhr.addEventListener('error', function () {
      reject(new Error('UPLOAD_NETWORK_ERROR'));
    });

    xhr.addEventListener('abort', function () {
      reject(new Error('UPLOAD_ABORTED'));
    });

    xhr.send(formData);
  });
};


---

Что будет после этого

Сейчас окно показывает просто:

UPLOAD_BAD_RESPONSE

После замены оно покажет реальную причину, например:

UPLOAD_BAD_RESPONSE. HTTP 413. Request Entity Too Large

или:

UPLOAD_BAD_RESPONSE. HTTP 200. POST Content-Length exceeds the limit...

или:

UPLOAD_BAD_RESPONSE. HTTP 504. Gateway Timeout


---

Скорее всего причина

Ты грузишь файл примерно:

189.8 МБ

Если компонент уже подняли до 1 ГБ, то дальше чаще всего режет PHP:

post_max_size
upload_max_filesize

Нужно минимум так:

upload_max_filesize = 1024M
post_max_size = 1024M
max_input_time = 300
max_execution_time = 300
memory_limit = 512M

И в Angie/Nginx:

client_max_body_size 1024m;

Но сначала замени метод apiUploadWithProgress, сделай Ctrl + F5, попробуй загрузить ещё раз и пришли уже новый текст ошибки из окна. Тогда будет точно видно, кто режет.