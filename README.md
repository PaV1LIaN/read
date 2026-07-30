Да. Сначала убедимся, что новая нода действительно мастер.

В pgAdmin на новой ноде выполни:

SELECT
    inet_server_addr()::text AS server_ip,
    current_user AS database_user,
    pg_is_in_recovery() AS is_replica,
    current_setting('transaction_read_only') AS transaction_read_only;

Нужно получить:

is_replica = false
transaction_read_only = off

После этого не запускай миграцию сразу. Сначала выполни диагностический файл:

Скачать диагностику миграции этапа 5

В pgAdmin:

ProjectsDB → Tools → Query Tool → Open File → Execute

Скрипт завершится ROLLBACK, поэтому базу не изменит.

Возможные результаты

Если во всех строках:

ok = true

тогда снова открывай:

/local/sitebuilder/tools/apply_stage13_migration.php

и запускай миграции один раз.

Если появится строка:

ok = false

пришли значения:

step_name
sqlstate
error_message

Если диагностический SQL снова сообщит про режим «только чтение», значит выбрана не мастер-нода.

/*
 * Безопасная диагностика миграции SiteBuilder stage 5.
 * Скрипт выполняет те же ключевые операции внутри транзакции,
 * записывает ошибки по шагам и в конце делает ROLLBACK.
 * Постоянных изменений в базе не оставляет.
 */

BEGIN;

SET LOCAL lock_timeout = '5s';
SET LOCAL statement_timeout = '60s';

CREATE TEMP TABLE sb_stage5_diag (
    step_no INTEGER NOT NULL,
    step_name TEXT NOT NULL,
    ok BOOLEAN NOT NULL,
    sqlstate TEXT NULL,
    error_message TEXT NULL
) ON COMMIT DROP;

DO $diag$
DECLARE
    v_detail TEXT;
    v_hint TEXT;
BEGIN
    /* Шаг 1: version для page */
    BEGIN
        ALTER TABLE sitebuilder.page
            ADD COLUMN IF NOT EXISTS version INTEGER;

        UPDATE sitebuilder.page
        SET version = 1
        WHERE version IS NULL OR version < 1;

        ALTER TABLE sitebuilder.page
            ALTER COLUMN version SET DEFAULT 1,
            ALTER COLUMN version SET NOT NULL;

        INSERT INTO sb_stage5_diag VALUES (1, 'page.version', TRUE, NULL, NULL);
    EXCEPTION WHEN OTHERS THEN
        GET STACKED DIAGNOSTICS
            v_detail = PG_EXCEPTION_DETAIL,
            v_hint = PG_EXCEPTION_HINT;
        INSERT INTO sb_stage5_diag VALUES (
            1,
            'page.version',
            FALSE,
            SQLSTATE,
            SQLERRM
                || CASE WHEN COALESCE(v_detail, '') <> '' THEN ' | DETAIL: ' || v_detail ELSE '' END
                || CASE WHEN COALESCE(v_hint, '') <> '' THEN ' | HINT: ' || v_hint ELSE '' END
        );
    END;

    /* Шаг 2: version для block */
    BEGIN
        ALTER TABLE sitebuilder.block
            ADD COLUMN IF NOT EXISTS version INTEGER;

        UPDATE sitebuilder.block
        SET version = 1
        WHERE version IS NULL OR version < 1;

        ALTER TABLE sitebuilder.block
            ALTER COLUMN version SET DEFAULT 1,
            ALTER COLUMN version SET NOT NULL;

        INSERT INTO sb_stage5_diag VALUES (2, 'block.version', TRUE, NULL, NULL);
    EXCEPTION WHEN OTHERS THEN
        GET STACKED DIAGNOSTICS
            v_detail = PG_EXCEPTION_DETAIL,
            v_hint = PG_EXCEPTION_HINT;
        INSERT INTO sb_stage5_diag VALUES (
            2,
            'block.version',
            FALSE,
            SQLSTATE,
            SQLERRM
                || CASE WHEN COALESCE(v_detail, '') <> '' THEN ' | DETAIL: ' || v_detail ELSE '' END
                || CASE WHEN COALESCE(v_hint, '') <> '' THEN ' | HINT: ' || v_hint ELSE '' END
        );
    END;

    /* Шаг 3: таблица ревизий */
    BEGIN
        CREATE TABLE IF NOT EXISTS sitebuilder.entity_revision (
            id BIGSERIAL PRIMARY KEY,
            site_id BIGINT NOT NULL,
            entity_type VARCHAR(16) NOT NULL,
            entity_id BIGINT NOT NULL,
            page_id BIGINT NULL,
            entity_version INTEGER NOT NULL,
            operation VARCHAR(32) NOT NULL,
            snapshot_json JSONB NOT NULL,
            created_by BIGINT NULL,
            created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
            restored_from_revision_id BIGINT NULL,
            CONSTRAINT entity_revision_type_chk
                CHECK (entity_type IN ('page', 'block')),
            CONSTRAINT entity_revision_version_chk
                CHECK (entity_version > 0)
        );

        INSERT INTO sb_stage5_diag VALUES (3, 'entity_revision.table', TRUE, NULL, NULL);
    EXCEPTION WHEN OTHERS THEN
        GET STACKED DIAGNOSTICS
            v_detail = PG_EXCEPTION_DETAIL,
            v_hint = PG_EXCEPTION_HINT;
        INSERT INTO sb_stage5_diag VALUES (
            3,
            'entity_revision.table',
            FALSE,
            SQLSTATE,
            SQLERRM
                || CASE WHEN COALESCE(v_detail, '') <> '' THEN ' | DETAIL: ' || v_detail ELSE '' END
                || CASE WHEN COALESCE(v_hint, '') <> '' THEN ' | HINT: ' || v_hint ELSE '' END
        );
    END;

    /* Шаг 4: индексы ревизий */
    BEGIN
        CREATE INDEX IF NOT EXISTS entity_revision_entity_idx
            ON sitebuilder.entity_revision (entity_type, entity_id, id DESC);

        CREATE INDEX IF NOT EXISTS entity_revision_site_idx
            ON sitebuilder.entity_revision (site_id, created_at DESC);

        CREATE INDEX IF NOT EXISTS entity_revision_page_idx
            ON sitebuilder.entity_revision (page_id, created_at DESC)
            WHERE page_id IS NOT NULL;

        INSERT INTO sb_stage5_diag VALUES (4, 'entity_revision.indexes', TRUE, NULL, NULL);
    EXCEPTION WHEN OTHERS THEN
        GET STACKED DIAGNOSTICS
            v_detail = PG_EXCEPTION_DETAIL,
            v_hint = PG_EXCEPTION_HINT;
        INSERT INTO sb_stage5_diag VALUES (
            4,
            'entity_revision.indexes',
            FALSE,
            SQLSTATE,
            SQLERRM
                || CASE WHEN COALESCE(v_detail, '') <> '' THEN ' | DETAIL: ' || v_detail ELSE '' END
                || CASE WHEN COALESCE(v_hint, '') <> '' THEN ' | HINT: ' || v_hint ELSE '' END
        );
    END;

    /* Шаг 5: исходные ревизии страниц */
    BEGIN
        INSERT INTO sitebuilder.entity_revision (
            site_id,
            entity_type,
            entity_id,
            page_id,
            entity_version,
            operation,
            snapshot_json,
            created_by,
            created_at
        )
        SELECT
            p.site_id,
            'page',
            p.id,
            p.id,
            p.version,
            'seed',
            jsonb_build_object(
                'id', p.id,
                'siteId', p.site_id,
                'title', p.title,
                'slug', p.slug,
                'parentId', COALESCE(p.parent_id, 0),
                'sort', p.sort,
                'status', p.status,
                'publishedAt', p.published_at,
                'createdBy', p.created_by,
                'createdAt', p.created_at,
                'updatedBy', p.updated_by,
                'updatedAt', p.updated_at,
                'version', p.version
            ),
            p.updated_by,
            COALESCE(p.updated_at, NOW())
        FROM sitebuilder.page p
        WHERE NOT EXISTS (
            SELECT 1
            FROM sitebuilder.entity_revision r
            WHERE r.entity_type = 'page'
              AND r.entity_id = p.id
        );

        INSERT INTO sb_stage5_diag VALUES (5, 'seed.page.revisions', TRUE, NULL, NULL);
    EXCEPTION WHEN OTHERS THEN
        GET STACKED DIAGNOSTICS
            v_detail = PG_EXCEPTION_DETAIL,
            v_hint = PG_EXCEPTION_HINT;
        INSERT INTO sb_stage5_diag VALUES (
            5,
            'seed.page.revisions',
            FALSE,
            SQLSTATE,
            SQLERRM
                || CASE WHEN COALESCE(v_detail, '') <> '' THEN ' | DETAIL: ' || v_detail ELSE '' END
                || CASE WHEN COALESCE(v_hint, '') <> '' THEN ' | HINT: ' || v_hint ELSE '' END
        );
    END;

    /* Шаг 6: исходные ревизии блоков */
    BEGIN
        INSERT INTO sitebuilder.entity_revision (
            site_id,
            entity_type,
            entity_id,
            page_id,
            entity_version,
            operation,
            snapshot_json,
            created_by,
            created_at
        )
        SELECT
            p.site_id,
            'block',
            b.id,
            b.page_id,
            b.version,
            'seed',
            jsonb_build_object(
                'id', b.id,
                'pageId', b.page_id,
                'type', b.type,
                'sort', b.sort,
                'content', b.content_json,
                'props', b.props_json,
                'createdBy', b.created_by,
                'createdAt', b.created_at,
                'updatedBy', b.updated_by,
                'updatedAt', b.updated_at,
                'version', b.version
            ),
            b.updated_by,
            COALESCE(b.updated_at, NOW())
        FROM sitebuilder.block b
        JOIN sitebuilder.page p ON p.id = b.page_id
        WHERE NOT EXISTS (
            SELECT 1
            FROM sitebuilder.entity_revision r
            WHERE r.entity_type = 'block'
              AND r.entity_id = b.id
        );

        INSERT INTO sb_stage5_diag VALUES (6, 'seed.block.revisions', TRUE, NULL, NULL);
    EXCEPTION WHEN OTHERS THEN
        GET STACKED DIAGNOSTICS
            v_detail = PG_EXCEPTION_DETAIL,
            v_hint = PG_EXCEPTION_HINT;
        INSERT INTO sb_stage5_diag VALUES (
            6,
            'seed.block.revisions',
            FALSE,
            SQLSTATE,
            SQLERRM
                || CASE WHEN COALESCE(v_detail, '') <> '' THEN ' | DETAIL: ' || v_detail ELSE '' END
                || CASE WHEN COALESCE(v_hint, '') <> '' THEN ' | HINT: ' || v_hint ELSE '' END
        );
    END;
END
$diag$;

/* Общая информация окружения. */
SELECT
    current_user AS database_user,
    current_setting('server_version') AS postgresql_version,
    to_regclass('sitebuilder.entity_revision') AS entity_revision_relation;

/* Основной результат диагностики. */
SELECT
    step_no,
    step_name,
    ok,
    sqlstate,
    error_message
FROM sb_stage5_diag
ORDER BY step_no;

/* Всегда отменяем все тестовые изменения. */
ROLLBACK