Да, это чинится в самом фреймворке.

Сейчас у нас redirect() возвращает пустой ответ с заголовком:

Location: /local/mvc_demo/form

Но если PHP уже что-то вывел до редиректа, например пробел, BOM или предупреждение, заголовок Location не отправляется. В итоге браузер получает пустой HTML — белый экран.

Сделаем безопасный redirect.


---

1. Замени /local/mvc/Core/Response.php

Полностью замени файл:

/local/mvc/Core/Response.php

на этот:

<?php

namespace Local\Mvc\Core;

/**
 * Response
 *
 * Ответ сервера пользователю.
 */
class Response
{
    private string $content;
    private int $status;
    private array $headers;

    public function __construct(string $content = '', int $status = 200, array $headers = [])
    {
        $this->content = $content;
        $this->status = $status;
        $this->headers = $headers;
    }

    public static function html(string $content, int $status = 200): self
    {
        return new self($content, $status, [
            'Content-Type' => 'text/html; charset=utf-8',
        ]);
    }

    public static function json(array $data, int $status = 200): self
    {
        return new self(
            json_encode($data, JSON_UNESCAPED_UNICODE),
            $status,
            [
                'Content-Type' => 'application/json; charset=utf-8',
            ]
        );
    }

    /**
     * Безопасный редирект.
     *
     * Если header Location отправится — браузер перейдёт сам.
     * Если header уже нельзя отправить — покажем HTML с переходом.
     */
    public static function redirect(string $url, int $status = 302): self
    {
        $safeUrl = htmlspecialchars($url, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');

        $content = '
            <!doctype html>
            <html lang="ru">
            <head>
                <meta charset="utf-8">
                <meta http-equiv="refresh" content="0;url=' . $safeUrl . '">
                <title>Переход...</title>
            </head>
            <body>
                <p>Переход...</p>
                <script>
                    window.location.href = ' . json_encode($url, JSON_UNESCAPED_UNICODE) . ';
                </script>
                <p>
                    Если переход не произошёл автоматически,
                    <a href="' . $safeUrl . '">нажмите сюда</a>.
                </p>
            </body>
            </html>
        ';

        return new self($content, $status, [
            'Location' => $url,
            'Content-Type' => 'text/html; charset=utf-8',
        ]);
    }

    public function header(string $name, string $value): self
    {
        $this->headers[$name] = $value;

        return $this;
    }

    public function send(): void
    {
        if (!headers_sent()) {
            http_response_code($this->status);

            foreach ($this->headers as $name => $value) {
                header($name . ': ' . $value, true);
            }
        }

        echo $this->content;
    }
}


---

2. Сбрось OPcache

В PHP-командной строке Битрикса:

opcache_reset();
echo 'OPcache reset OK';


---

3. Проверь форму снова

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/form

Заполни форму и отправь.

Теперь после отправки должно перекинуть обратно на:

/local/mvc_demo/form

и показать зелёное flash-сообщение.


---

Если снова белый экран

Тогда надо временно включить диагностику именно в /local/mvc_demo/index.php.

В самое начало после <?php добавь:

ini_set('display_errors', '1');
ini_set('display_startup_errors', '1');
error_reporting(E_ALL);

register_shutdown_function(function () {
    $error = error_get_last();

    if ($error !== null) {
        echo '<pre style="background:#300;color:#fff;padding:20px;">';
        print_r($error);
        echo '</pre>';
    }
});

Но сначала попробуй замену Response.php. Скорее всего, белый экран уйдёт.