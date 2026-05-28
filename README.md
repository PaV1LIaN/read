Идём дальше. Сейчас сделаем редирект по имени маршрута.

Сейчас в контроллере мы можем писать так:

return $this->redirect('/local/mvc_demo/form');

Но это плохо, потому что контроллер знает точный URL проекта.

Хотим так:

return $this->redirectRoute('form.index');

То есть:

Не важно, какой URL у формы.
Главное — есть маршрут с именем form.index.


---

1. Обнови /local/mvc/Core/Controller.php

Открой файл:

/local/mvc/Core/Controller.php

Найди метод:

protected function redirect(string $url): Response
{
    return Response::redirect($url);
}

Сразу после него добавь:

/**
 * Собрать URL по имени маршрута.
 *
 * Например:
 * $this->route('admin.users.show', ['id' => 5])
 */
protected function route(string $name, array $params = [], array $query = []): string
{
    return App::route($name, $params, $query);
}

/**
 * Redirect по имени маршрута.
 *
 * Например:
 * return $this->redirectRoute('form.index');
 */
protected function redirectRoute(string $name, array $params = [], array $query = []): Response
{
    return $this->redirect($this->route($name, $params, $query));
}

Теперь любой контроллер может делать:

$this->route(...)
$this->redirectRoute(...)


---

2. Проверь имя маршрута формы

Открой:

/local/mvc_demo/routes.php

Найди маршрут формы:

$router->get('/form', [FormController::class, 'index']);

Замени на именованный:

$router->get('/form', [FormController::class, 'index'], [], 'form.index');

POST можно тоже назвать:

$router->post('/form/send', [FormController::class, 'send'], ['csrf'], 'form.send');

Должно быть так:

$router->get('/form', [FormController::class, 'index'], [], 'form.index');

$router->post('/form/send', [FormController::class, 'send'], ['csrf'], 'form.send');


---

3. Обнови /local/mvc_demo/Controllers/FormController.php

В методе send() у нас сейчас есть:

return $this->redirectBack('/form');

Замени оба таких места на:

return $this->redirectRoute('form.index');

Полный метод send() должен быть такой:

public function send(): Response
{
    $data = [
        'name' => trim((string)$this->request->post('name', '')),
        'message' => trim((string)$this->request->post('message', '')),
    ];

    $errors = [];

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

        return $this->redirectRoute('form.index');
    }

    Flash::success('Форма успешно отправлена. Имя: ' . $data['name']);

    return $this->redirectRoute('form.index');
}


---

4. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Проверь форму:

https://bitrix24-stage.gaz.ru/local/mvc_demo/form

Сценарии те же:

1. Отправь с ошибками.
   Должно вернуть на форму, показать ошибки и сохранить поля.

2. Отправь нормально.
   Должно вернуть на форму, показать зелёное сообщение и очистить поля.


---

Что мы сделали

Раньше контроллер знал конкретный адрес:

return $this->redirect('/local/mvc_demo/form');

Теперь контроллер знает только имя маршрута:

return $this->redirectRoute('form.index');

Это правильнее.

Потом в sitebuilder будет красиво:

return $this->redirectRoute('sites.index');

return $this->redirectRoute('sites.edit', [
    'siteId' => $siteId,
]);

return $this->redirectRoute('pages.edit', [
    'siteId' => $siteId,
    'pageId' => $pageId,
]);

Главная мысль:

URL может измениться.
Имя маршрута остаётся.

Дальше логично сделать короткую функцию для ссылок во views, чтобы там тоже не писать \Local\Mvc\Core\App::route(...) полностью.