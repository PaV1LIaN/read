<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Создание заявки</title>
</head>
<body>
    <h1>Создание заявки</h1>   
    <form id="requestInvoice"action="">
        <div>
            <label id="invoiceTheme" for="labelText">Тема заявки</label>
            <input id="invoiceTheme" type="text">
        </div>
        <div id="block">
            <label for="priority">Приоритет</label>
            <input id="priorityLow" name="1" type="radio"> low
            <input id="priorityMedium" name="1" type="radio"> medium
            <input id="priorityHigh" name="1" type="radio"> High
        </div>
        <div>
            <label for="deviceCount">Количество устройств</label>
            <input id="deviceCount" type="number">
        </div>
        <button id="buttonResult" type="submit">Создать заявку</button>
    </form>
    <p id="result">Здесь появится результат</p>
</body>
</html>
