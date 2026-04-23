console
Error: Call to undefined method Bitrix\Disk\Folder::getPath()
    at DiskComponent.loadFolder (script.js:174:15)
    at async DiskComponent.init (script.js:70:7)

response 
{
    "ok": true,
    "data": {
        "siteId": 7,
        "pageId": 7,
        "blockId": 6,
        "settings": {
            "title": "Файлы",
            "rootMode": "site",
            "rootFolderId": null,
            "viewMode": "table",
            "allowUpload": true,
            "allowCreateFolder": true,
            "allowRename": true,
            "allowDelete": true,
            "allowDownload": true,
            "showSearch": true,
            "showBreadcrumbs": false,
            "defaultSort": "updatedAt",
            "defaultSortDirection": "desc",
            "allowedExtensions": [],
            "maxFileSize": 52428800,
            "permissionMode": "inherit_site",
            "useSiteRootFallback": true
        },
        "permissions": {
            "canView": true,
            "canUpload": true,
            "canCreateFolder": true,
            "canRename": true,
            "canDelete": true,
            "canDownload": true,
            "canManageAccess": true,
            "canEditSettings": true
        },
        "rootFolderId": 324,
        "currentFolderId": 324,
        "rootSource": "site"
    },
    "meta": [],
    "error": null,
    "message": ""
}
