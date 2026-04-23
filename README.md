BAD_SESSID сейчас почти наверняка из-за того, что твой disk ходит в api.php JSON-запросами, а серверная CSRF-проверка, скорее всего, заточена под обычный $_POST/$_REQUEST и check_bitrix_sessid().

То есть:

sessid у тебя уже передается

но валидатор его проверяет не тем способом


Самый надежный фикс — переделать DiskCsrf::validateFromRequest(), чтобы он:

1. умел читать sessid и из JSON body


2. сравнивал его с bitrix_sessid()




---

Что заменить

Файл

/local/sitebuilder/components/disk/lib/DiskCsrf.php

Замени его содержимое на это:

<?php

class DiskCsrf
{
    public static function validateFromRequest(): void
    {
        $sessid = self::extractSessid();

        if ($sessid === '') {
            throw new RuntimeException('EMPTY_SESSID');
        }

        $currentSessid = (string)bitrix_sessid();

        if ($currentSessid === '') {
            throw new RuntimeException('BAD_SESSID');
        }

        if (!hash_equals($currentSessid, $sessid)) {
            throw new RuntimeException('BAD_SESSID');
        }
    }

    protected static function extractSessid(): string
    {
        if (!empty($_POST['sessid'])) {
            return trim((string)$_POST['sessid']);
        }

        if (!empty($_REQUEST['sessid'])) {
            return trim((string)$_REQUEST['sessid']);
        }

        $raw = file_get_contents('php://input');
        if (!is_string($raw) || trim($raw) === '') {
            return '';
        }

        $decoded = json_decode($raw, true);
        if (is_array($decoded) && !empty($decoded['sessid'])) {
            return trim((string)$decoded['sessid']);
        }

        return '';
    }
}


---

Почему это должно помочь

У тебя disk/script.js отправляет sessid в JSON:

body: JSON.stringify(payload)

А стандартная битриксовая проверка часто ждет его в обычном POST-поле.
После этого фикса DiskCsrf будет корректно брать sessid и из JSON body тоже.


---

Что сделать после замены

1. сохранить DiskCsrf.php


2. сделать Ctrl + F5


3. снова открыть страницу с блоком disk


4. посмотреть bootstrap response




---

Что должно быть дальше

После этого BAD_SESSID должен уйти.

И тогда уже появится либо:

нормальный ok: true

либо следующая реальная ошибка по:

правам

root папке

context

Bitrix Disk



Если после замены появится новая ошибка, пришли bootstrap response целиком.