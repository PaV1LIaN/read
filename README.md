Да, идём поэтапно. Начинаем с безопасной основы: при создании сайта создаём связанную группу Битрикс24 и сохраняем её ID в таблицу сайта, но текущие права sitebuilder.site_user_access пока не ломаем.


---

Этап 1. Добавь поля в таблицу сайтов

Выполни в PostgreSQL:

ALTER TABLE sitebuilder.sites
ADD COLUMN IF NOT EXISTS bitrix_group_id integer DEFAULT NULL;

ALTER TABLE sitebuilder.sites
ADD COLUMN IF NOT EXISTS bitrix_group_created_by integer DEFAULT NULL;

ALTER TABLE sitebuilder.sites
ADD COLUMN IF NOT EXISTS bitrix_group_created_at timestamp DEFAULT NULL;

CREATE UNIQUE INDEX IF NOT EXISTS uq_sitebuilder_sites_bitrix_group_id
ON sitebuilder.sites(bitrix_group_id)
WHERE bitrix_group_id IS NOT NULL;


---

Этап 2. Создай сервис создания группы

Создай файл:

/local/sitebuilder/lib/SiteBitrixGroupService.php

Код:

<?php

use Bitrix\Main\Loader;

class SiteBitrixGroupService
{
    public static function createForSite(array $site, int $ownerUserId): int
    {
        if ($ownerUserId <= 0) {
            throw new RuntimeException('EMPTY_OWNER_USER_ID');
        }

        if (!Loader::includeModule('socialnetwork')) {
            throw new RuntimeException('SOCIALNETWORK_MODULE_NOT_INSTALLED');
        }

        if (!class_exists('CSocNetGroup')) {
            throw new RuntimeException('CSocNetGroup_NOT_FOUND');
        }

        $siteId = (int)($site['id'] ?? 0);
        $siteName = trim((string)($site['name'] ?? ''));

        if ($siteId <= 0) {
            throw new RuntimeException('EMPTY_SITE_ID');
        }

        if ($siteName === '') {
            $siteName = 'Сайт #' . $siteId;
        }

        $groupName = self::buildGroupName($siteName, $siteId);
        $subjectId = self::resolveSubjectId();

        $fields = [
            'SITE_ID' => self::getSiteId(),
            'NAME' => $groupName,
            'DESCRIPTION' => 'Рабочая группа сайта SiteBuilder: ' . $siteName,
            'VISIBLE' => 'N',
            'OPENED' => 'N',
            'PROJECT' => 'N',
            'SUBJECT_ID' => $subjectId,
            'INITIATE_PERMS' => self::ownerRole(),
            'SPAM_PERMS' => self::ownerRole(),
        ];

        $groupId = (int)\CSocNetGroup::CreateGroup(
            $ownerUserId,
            $fields,
            false
        );

        if ($groupId <= 0) {
            $message = self::getLastBitrixError();
            throw new RuntimeException('BITRIX_GROUP_CREATE_ERROR' . ($message !== '' ? ': ' . $message : ''));
        }

        return $groupId;
    }

    protected static function buildGroupName(string $siteName, int $siteId): string
    {
        $siteName = trim($siteName);

        if ($siteName === '') {
            $siteName = 'Сайт #' . $siteId;
        }

        return 'SiteBuilder: ' . $siteName;
    }

    protected static function resolveSubjectId(): int
    {
        if (!class_exists('CSocNetGroupSubject')) {
            return 1;
        }

        $siteId = self::getSiteId();

        $rs = \CSocNetGroupSubject::GetList(
            ['SORT' => 'ASC', 'NAME' => 'ASC'],
            ['SITE_ID' => $siteId],
            false,
            ['nTopCount' => 1],
            ['ID']
        );

        if ($row = $rs->Fetch()) {
            $id = (int)($row['ID'] ?? 0);
            if ($id > 0) {
                return $id;
            }
        }

        $rs = \CSocNetGroupSubject::GetList(
            ['SORT' => 'ASC', 'NAME' => 'ASC'],
            [],
            false,
            ['nTopCount' => 1],
            ['ID']
        );

        if ($row = $rs->Fetch()) {
            $id = (int)($row['ID'] ?? 0);
            if ($id > 0) {
                return $id;
            }
        }

        return 1;
    }

    protected static function getSiteId(): string
    {
        if (defined('SITE_ID') && SITE_ID) {
            return (string)SITE_ID;
        }

        return 's1';
    }

    protected static function ownerRole(): string
    {
        return defined('SONET_ROLES_OWNER') ? SONET_ROLES_OWNER : 'A';
    }

    protected static function getLastBitrixError(): string
    {
        global $APPLICATION;

        if (is_object($APPLICATION) && method_exists($APPLICATION, 'GetException')) {
            $exception = $APPLICATION->GetException();
            if ($exception && method_exists($exception, 'GetString')) {
                return trim((string)$exception->GetString());
            }
        }

        return '';
    }
}


---

Этап 3. Подключить при создании сайта

Теперь нужно править файл создания сайта:

/local/sitebuilder/api/handlers/site.php

В начале файла, рядом с другими require_once, добавь:

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/SiteBitrixGroupService.php';

После того как сайт уже создан и у тебя есть $siteId, добавь создание группы:

$bitrixGroupId = 0;
$bitrixGroupError = '';

try {
    $bitrixGroupId = SiteBitrixGroupService::createForSite([
        'id' => $siteId,
        'name' => $name,
    ], (int)$USER->GetID());

    if ($bitrixGroupId > 0) {
        sb_db_execute("
            UPDATE sitebuilder.sites
            SET
                bitrix_group_id = :bitrix_group_id,
                bitrix_group_created_by = :created_by,
                bitrix_group_created_at = now()
            WHERE id = :site_id
        ", [
            ':bitrix_group_id' => $bitrixGroupId,
            ':created_by' => (int)$USER->GetID(),
            ':site_id' => $siteId,
        ]);
    }
} catch (Throwable $e) {
    $bitrixGroupError = $e->getMessage();
}

И в ответ API желательно добавить в data или meta:

'bitrixGroupId' => $bitrixGroupId,
'bitrixGroupError' => $bitrixGroupError,


---

Важный момент

Пока группа будет создаваться как связанное рабочее пространство сайта, но права конструктора остаются в:

sitebuilder.site_user_access

На следующем этапе сделаем связку:

участники группы Битрикс24 → роли sitebuilder

Например:

владелец группы       → site_admin
модератор группы     → editor
участник группы      → viewer

Сначала лучше внедрить создание группы и проверить, что в sitebuilder.sites.bitrix_group_id реально записывается ID.