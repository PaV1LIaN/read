<pre>=== CONNECTION ===
Array
(
    [DB_NAME] => bx
    [CURRENT_SCHEMA] => public
    [DB_USER] => bx_user
    [SEARCH_PATH] => "$user", public
    [SERVER_IP] => 192.168.7.110
    [SERVER_PORT] => 5432
)

=== FIND MVC TABLES ===
Array
(
    [0] => Array
        (
            [TABLE_SCHEMA] => mvc
            [TABLE_NAME] => mvc_demo_notes
        )

)

=== FIND NOTES TABLES ===
Array
(
    [0] => Array
        (
            [TABLE_SCHEMA] => mvc
            [TABLE_NAME] => mvc_demo_notes
        )

    [1] => Array
        (
            [TABLE_SCHEMA] => public
            [TABLE_NAME] => b_booking_booking_note
        )

    [2] => Array
        (
            [TABLE_SCHEMA] => public
            [TABLE_NAME] => b_crm_timeline_note
        )

)

=== TO_REGCLASS ===
Array
(
    [MVC_TABLE] => mvc.mvc_demo_notes
    [PUBLIC_TABLE] => 
)

=== DATA CHECK ===
Array
(
    [0] => Array
        (
            [ID] => 2
            [TITLE] => Тестовая заметка
            [BODY] => Проверка записи в PostgreSQL
            [CREATED_AT] => Bitrix\Main\Type\DateTime Object
                (
                    [value:protected] => DateTime Object
                        (
                            [date] => 2026-05-29 14:28:26.000000
                            [timezone_type] => 3
                            [timezone] => Europe/Moscow
                        )

                    [userTimeEnabled:protected] => 1
                )

            [UPDATED_AT] => Bitrix\Main\Type\DateTime Object
                (
                    [value:protected] => DateTime Object
                        (
                            [date] => 2026-05-29 14:28:26.000000
                            [timezone_type] => 3
                            [timezone] => Europe/Moscow
                        )

                    [userTimeEnabled:protected] => 1
                )

        )

    [1] => Array
        (
            [ID] => 1
            [TITLE] => test
            [BODY] => test
            [CREATED_AT] => Bitrix\Main\Type\DateTime Object
                (
                    [value:protected] => DateTime Object
                        (
                            [date] => 2026-05-29 14:25:00.000000
                            [timezone_type] => 3
                            [timezone] => Europe/Moscow
                        )

                    [userTimeEnabled:protected] => 1
                )

            [UPDATED_AT] => Bitrix\Main\Type\DateTime Object
                (
                    [value:protected] => DateTime Object
                        (
                            [date] => 2026-05-29 14:25:00.000000
                            [timezone_type] => 3
                            [timezone] => Europe/Moscow
                        )

                    [userTimeEnabled:protected] => 1
                )

        )

)
</pre>
