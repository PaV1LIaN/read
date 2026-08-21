<!DOCTYPE HTML5>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Создание заявки</title>
</head>
<body>
    <h1>Создание заявки</h1>   
    <form id="requestInvoice"action="">
        <div>
            <label for="invoiceTheme">Тема заявки</label>
            <input id="invoiceTheme" type="text">
        </div>
        <div>
            <label>Приоритет</label>
            <input id="priorityLow" name="priority" type="radio" value="low"> 
            <label for="priorityLow">low</label>
            <input id="priorityNormal" name="priority" type="radio" value="normal"> 
            <label for="priorityNormal">normal</label>
            <input id="priorityHigh" name="priority" type="radio" value="high">
            <label for="priorityHigh">high</label>
        </div>
        <div>
            <label for="deviceCount">Количество устройств</label>
            <input id="deviceCount" type="number">
        </div>
        <button type="submit">Создать заявку</button>
    </form>
    <p id="result">Здесь появится результат</p>
</body>
</html>
