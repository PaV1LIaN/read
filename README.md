Идём дальше. Сделаем удобный метод:

$this->redirectBack('/form')

Он будет означать:

Верни пользователя туда, откуда он пришёл.
Если не знаешь откуда — верни на /form.

Это пригодится везде:

формы
создание сайта
сохранение страницы
удаление записи
ошибки доступа


---

1. Обновляем /local/mvc/Core/Controller.php

Открой файл:

/local/mvc/Core/Controller.php

Найди метод:

protected function redirect(string $url): Response
{
    return Response::redirect($url);
}

Сразу после него добавь:

/**
 * Вернуть пользователя назад.
 *
 * Например:
 * return $this->redirectBack('/form');
 *
 * Если есть HTTP_REFERER — вернём туда.
 * Если нет — вернём на fallback внутри текущего проекта.
 */
protected function redirectBack(string $fallbackPath = '/'): Response
{
    $referer = (string)$this->request->server('HTTP_REFERER', '');

    if ($this->isSafeRedirectUrl($referer)) {
        return $this->redirect($referer);
    }

    return $this->redirect($this->projectUrlPath($fallbackPath));
}

/**
 * Собрать URL внутри текущего проекта.
 *
 * Например:
 * projectUrlPath('/form')
 *
 * вернёт:
 * /local/mvc_demo/form
 */
protected function projectUrlPath(string $path = '/'): string
{
    $base = $this->projectUrl();

    $path = trim($path);

    if ($path === '' || $path === '/') {
        return $base . '/';
    }

    return $base . '/' . trim($path, '/');
}

/**
 * Проверяем, безопасный ли URL для редиректа.
 *
 * Нельзя слепо редиректить на любой адрес,
 * потому что кто-то может подсунуть внешний сайт.
 */
private function isSafeRedirectUrl(string $url): bool
{
    $url = trim($url);

    if ($url === '') {
        return false;
    }

    /**
     * Локальный путь безопасен:
     * /local/mvc_demo/form
     */
    if (str_starts_with($url, '/')) {
        return true;
    }

    /**
     * Полный URL разрешаем только если host такой же,
     * как у текущего портала.
     */
    $currentHost = (string)$this->request->server('HTTP_HOST', '');

    $parts = parse_url($url);

    if (!is_array($parts)) {
        return false;
    }

    $urlHost = (string)($parts['host'] ?? '');

    if ($urlHost === '' || $currentHost === '') {
        return false;
    }

    return strcasecmp($urlHost, $currentHost) === 0;
}


---

2. Обновляем /local/mvc_demo/Controllers/FormController.php

Теперь в конце метода send() у нас было:

return $this->redirect('/local/mvc_demo/form');

Замени оба таких места на:

return $this->redirectBack('/form');

Полный метод send() должен стать таким:

public function send(): Response
{
    $data = [
        'name' => trim((string)$this->request->post('name', '')),
        'message' => trim((string)$this->request->post('message', '')),
    ];

    $errors = [];

    if (function_exists('check_bitrix_sessid') && !check_bitrix_sessid()) {
        $errors[] = 'Ошибка безопасности: неверный sessid.';
    }

    $validator = Validator::make($data)
        ->required('name', 'Введите имя.')
        ->min('name', 2, 'Имя должно быть не короче 2 символов.')
        ->max('name', 100, 'Имя должно быть не длиннее 100 символов.')
        ->required('message', 'Введите сообщение.')
        ->min('message', 5, 'Сообщение должно быть не короче 5 символов.')
        ->max('message', 1000, 'Сообщение должно быть не длиннее 1000 символов.');

    if ($validator->fails()) {
        $errors = array_merge($errors, $validator->errorList());
    }

    if (!empty($errors)) {
        Flash::old($data);

        foreach ($errors as $error) {
            Flash::error($error);
        }

        return $this->redirectBack('/form');
    }

    Flash::success('Форма успешно отправлена. Имя: ' . $data['name']);

    return $this->redirectBack('/form');
}


---

3. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Потом открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/form

Проверь 2 сценария:

1. Отправь форму с ошибками.
   Должно вернуть обратно на /form, показать ошибки и сохранить поля.

2. Отправь форму правильно.
   Должно вернуть обратно на /form, показать зелёное сообщение и очистить поля.


---

Что мы сделали

Раньше контроллер знал точный адрес проекта:

return $this->redirect('/local/mvc_demo/form');

Это плохо, потому что если проект переедет:

/local/mvc_demo

например в:

/local/sitebuilder

придётся искать и менять ссылки.

Теперь лучше:

return $this->redirectBack('/form');

Контроллер говорит:

Верни назад.
Если назад нельзя — верни на страницу формы текущего проекта.

Это уже поведение фреймворка, а не конкретного проекта.