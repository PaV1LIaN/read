SELECT
    a.id AS attempt_id,
    s.id AS survey_id,
    s.title AS survey_title,

    a.user_id,
    a.user_fio,
    COALESCE(NULLIF(a.department_name, ''), 'Не указано')
        AS department_name,

    a.status,
    a.started_at,
    a.completed_at,

    qg.id AS group_id,
    COALESCE(qg.title, 'Без группы') AS group_title,

    q.id AS question_id,
    q.question_text,
    q.question_type,

    ans.id AS answer_id,
    ans.option_id,
    o.option_text,

    CASE
        WHEN ans.option_id IS NOT NULL
             AND NULLIF(TRIM(ans.answer_text), '') IS NOT NULL
            THEN o.option_text || ': ' || ans.answer_text

        WHEN ans.option_id IS NOT NULL
            THEN o.option_text

        ELSE ans.answer_text
    END AS answer_value,

    ans.created_at AS answer_created_at,
    ans.updated_at AS answer_updated_at

FROM survey.attempts a

INNER JOIN survey.surveys s
    ON s.id = a.survey_id

INNER JOIN survey.answers ans
    ON ans.attempt_id = a.id
   AND ans.is_active = TRUE

INNER JOIN survey.questions q
    ON q.id = ans.question_id

LEFT JOIN survey.question_groups qg
    ON qg.id = q.group_id

LEFT JOIN survey.options o
    ON o.id = ans.option_id

WHERE a.status = 'completed'

ORDER BY
    s.id,
    a.completed_at DESC,
    a.id,
    COALESCE(qg.sort_order, 999999),
    q.sort_order,
    o.sort_order,
    ans.id;