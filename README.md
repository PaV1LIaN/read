Да, это из-за того, что в правой колонке мало ширины, а блок выдачи прав сделан в одну строку: Пользователь + Роль. Лучше сделать их друг под другом.

В editor.php в <style> добавь в самый конец:

/* Исправление блока прав пользователей в узкой правой колонке */
.sb-access-form {
    display: grid;
    grid-template-columns: 1fr !important;
    gap: 12px;
}

.sb-access-form .sb-field {
    min-width: 0;
}

.sb-access-form .sb-input,
.sb-access-form .sb-select {
    width: 100%;
    max-width: 100%;
    box-sizing: border-box;
}

/* Чтобы список пользователей не выпирал */
.sb-access-list {
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.sb-access-item {
    display: grid;
    grid-template-columns: minmax(0, 1fr) auto auto;
    gap: 8px;
    align-items: center;
    max-width: 100%;
    overflow: hidden;
}

.sb-access-item__main {
    min-width: 0;
    overflow: hidden;
}

.sb-access-item__name,
.sb-access-item__meta {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}

.sb-access-item .sb-role-badge {
    white-space: nowrap;
}

.sb-access-item .sb-btn {
    white-space: nowrap;
}

@media (max-width: 1500px) {
    .sb-access-item {
        grid-template-columns: minmax(0, 1fr) auto;
    }

    .sb-access-item .sb-btn-danger {
        grid-column: 1 / -1;
        justify-self: flex-start;
    }
}

Если после этого роль в списке всё равно будет наезжать, значит в renderAccessList() используются другие классы. Тогда пришли кусок функции renderAccessList(), и я подгоню CSS точно под твою разметку.