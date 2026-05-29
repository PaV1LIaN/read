Отлично, теперь всё понятно.

Таблица реально лежит здесь:

server_ip: 192.168.7.110
database: bx
schema: mvc
table: mvc_demo_notes

То есть в pgAdmin надо смотреть именно:

Servers
  → 192.168.7.110
    → Databases
      → bx
        → Schemas
          → mvc
            → Tables
              → mvc_demo_notes

Если в дереве pgAdmin её не видно — сделай правой кнопкой:

Schemas → Refresh
mvc → Refresh
Tables → Refresh

А если всё равно не видно, значит в pgAdmin открыт не тот сервер/подключение. Данные точно есть, PHP их видит.


---

Теперь важный момент: Битрикс отдаёт поля из PostgreSQL в верхнем регистре:

ID
TITLE
BODY
CREATED_AT
UPDATED_AT

А мы в модели работаем с нижним регистром:

id
title
body
created_at
updated_at

Чтобы дальше не путаться, сделаем нормально: модель Note сама будет приводить данные к нормальному Laravel-like виду.

1. Замени /local/mvc_demo/Models/Note.php

<?php

namespace Local\MvcDemo\Models;

use Local\Mvc\Core\Model;

class Note extends Model
{
    protected static string $table = 'mvc.mvc_demo_notes';

    protected static string $primaryKey = 'id';

    protected static array $fillable = [
        'title',
        'body',
        'created_at',
        'updated_at',
    ];

    public static function latest(int $limit = 20): array
    {
        $rows = self::query()
            ->orderBy('id', 'desc')
            ->limit($limit)
            ->get();

        return array_map([self::class, 'normalize'], $rows);
    }

    public static function findNormalized(int $id): ?array
    {
        $row = self::find($id);

        if (!$row) {
            return null;
        }

        return self::normalize($row);
    }

    public static function normalize(array $row): array
    {
        return [
            'id' => $row['id'] ?? $row['ID'] ?? null,
            'title' => $row['title'] ?? $row['TITLE'] ?? '',
            'body' => $row['body'] ?? $row['BODY'] ?? '',
            'created_at' => $row['created_at'] ?? $row['CREATED_AT'] ?? '',
            'updated_at' => $row['updated_at'] ?? $row['UPDATED_AT'] ?? '',
        ];
    }
}

Теперь view всегда будет получать нормальные ключи:

$note['id']
$note['title']
$note['body']


---

2. Делаем Laravel-like FormRequest для заметок

Создай файл:

/local/mvc_demo/Requests/StoreNoteRequest.php

<?php

namespace Local\MvcDemo\Requests;

use Local\Mvc\Core\FormRequest;

class StoreNoteRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'title' => ['required', 'min:2', 'max:255'],
            'body' => ['max:2000'],
        ];
    }

    public function messages(): array
    {
        return [
            'title.required' => 'Введите название заметки.',
            'title.min' => 'Название должно быть не короче 2 символов.',
            'title.max' => 'Название должно быть не длиннее 255 символов.',
            'body.max' => 'Текст заметки должен быть не длиннее 2000 символов.',
        ];
    }

    public function redirectRoute(): ?string
    {
        return 'notes.index';
    }
}


---

3. Обнови /local/mvc_demo/Controllers/NoteController.php

Полностью замени файл:

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Flash;
use Local\Mvc\Core\Response;
use Local\MvcDemo\Models\Note;
use Local\MvcDemo\Requests\StoreNoteRequest;

class NoteController extends Controller
{
    public function index(): Response
    {
        return $this->render('notes/index', [
            'title' => 'Заметки',
            'notes' => Note::latest(20),
        ]);
    }

    public function store(StoreNoteRequest $request): Response
    {
        $data = $request->validated();

        Note::create([
            'title' => $data['title'],
            'body' => $data['body'],
            'created_at' => date('Y-m-d H:i:s'),
            'updated_at' => date('Y-m-d H:i:s'),
        ]);

        Flash::success('Заметка создана.');

        return redirect()->route('notes.index');
    }

    public function delete(string $id): Response
    {
        Note::deleteById((int)$id);

        Flash::success('Заметка удалена.');

        return redirect()->route('notes.index');
    }
}

Теперь контроллер стал ближе к Laravel:

public function store(StoreNoteRequest $request): Response
{
    $data = $request->validated();
}

Если данные плохие — контроллер вообще не запустится, FormRequest сам вернёт назад с ошибками.


---

4. Проверь view

В /local/mvc_demo/Views/notes/index.php теперь можно спокойно использовать нижний регистр:

$note['id']
$note['title']
$note['body']
$note['created_at']

Потому что модель Note::latest() уже нормализует данные.


---

5. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/notes

Проверь:

1. Отправь пустое название.
   Должна появиться ошибка.

2. Введи нормальную заметку.
   Она должна сохраниться.

3. Проверь в PHP-командной строке:

use Bitrix\Main\Application;

$rows = Application::getConnection()->query("
    SELECT *
    FROM mvc.mvc_demo_notes
    ORDER BY id DESC
")->fetchAll();

echo '<pre>';
print_r($rows);
echo '</pre>';


---

Что мы сейчас улучшили:

NoteController — стал тоньше
StoreNoteRequest — отвечает за валидацию
Note model — приводит данные к удобному виду

Это уже очень Laravel-like подход.