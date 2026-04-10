Отлично, причина найдена точно:

в page.php вызывается sb_next_id(), но эта функция не подключена в текущем bootstrap/наборах lib.

То есть проблема не в кнопке и не в JS.
Нужно просто добавить функцию sb_next_id() в общий lib, который точно подключается API.


---

Что сделать

Открой файл:

/local/sitebuilder/lib/helpers.php

И добавь в него этот блок кода:

if (!function_exists('sb_next_id')) {
    function sb_next_id(array $rows, string $key = 'id'): int
    {
        $max = 0;

        foreach ($rows as $row) {
            $value = (int)($row[$key] ?? 0);
            if ($value > $max) {
                $max = $value;
            }
        }

        return $max + 1;
    }
}

Лучше вставить его ближе к верхней части файла, рядом с другими общими helper-функциями.


---

Почему это сработает

page.php использует:

sb_next_id($pages, 'id')

а функции нет в загруженных helper-ах.
После добавления:

page.create заработает

page.duplicate тоже заработает

и другие handler-ы, если они тоже используют sb_next_id, тоже перестанут падать



---

Что делать дальше

1.

Добавь функцию в:

/local/sitebuilder/lib/helpers.php

2.

Сохрани файл

3.

Обнови editor.php

4.

Снова нажми Создать


---

Если после этого будет новая ошибка

Пришли следующий Response, и я добью до рабочего состояния.