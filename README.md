Отлично. Идём по правильному пути: сначала parentId, потом древовидное левое меню.

Раз файлы, которые ты раньше загружал сюда, уже недоступны, работаем по твоей текущей структуре из чата. Если захочешь, чтобы я сверил что-то с конкретным файлом целиком, его нужно будет загрузить заново.

Шаг 1. Добавляем parentId в страницы

Нужно изменить api.php в двух местах:

page.create

page.update


И задать дефолт parentId = 0.


---

1. В page.create

Найди action:

if ($action === 'page.create') {

И внутри него, рядом с чтением полей страницы, добавь:

$parentId = (int)($_POST['parentId'] ?? 0);


---

Потом перед созданием $page добавь проверку:

if ($parentId > 0) {
    $parent = sb_find_page($parentId);
    if (!$parent || (int)($parent['siteId'] ?? 0) !== $siteId) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'PARENT_PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }
}


---

А в сам массив страницы добавь:

'parentId' => $parentId,

То есть страница должна содержать это поле.


---

2. В page.update

Найди action:

if ($action === 'page.update') {

И добавь чтение поля:

$parentId = (int)($_POST['parentId'] ?? 0);


---

Потом валидацию перед сохранением:

if ($parentId > 0) {
    if ($parentId === $id) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'PAGE_CANNOT_BE_PARENT_OF_ITSELF'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $parent = sb_find_page($parentId);
    if (!$parent || (int)($parent['siteId'] ?? 0) !== $siteId) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'PARENT_PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }
}


---

И при обновлении страницы записывай:

$page['parentId'] = $parentId;

или если у тебя цикл по массиву страниц:

$p['parentId'] = $parentId;


---

3. Если старые страницы уже существуют

Ничего страшного. Для старых записей поле может отсутствовать — мы будем считать это как 0.


---

Шаг 2. Что нужно сделать в UI страниц

Теперь нужно в месте, где ты:

создаёшь страницу,

редактируешь страницу,


добавить выбор родителя.

То есть в форме страницы нужен select:

Нет родителя

Главная

О компании

Услуги

и т.д.


Но пока давай сначала сделаем серверную часть, чтобы всё было готово.


---

Что проверить после правок

После изменений в api.php:

1. создай страницу с parentId = 0


2. создай вторую страницу с parentId = id первой


3. проверь, что в pages.json у второй страницы появился parentId



Пример ожидаемой записи:

{
  "id": 2,
  "siteId": 1,
  "title": "Команда",
  "slug": "team",
  "parentId": 1
}


---

Дальше

Как только это будет внесено, следующий шаг — я дам тебе:

что добавить в index.php или где у тебя редактируются страницы,

чтобы можно было выбирать родителя из интерфейса.