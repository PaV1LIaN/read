Как я понял при разработке мы добавляли таблицы, но в дальнейшем некоторые из них не использовали вообще

TABLES IN SCHEMA sitebuilder:

Array
(
    [0] => Array
        (
            [table_schema] => sitebuilder
            [table_name] => access
        )

    [1] => Array
        (
            [table_schema] => sitebuilder
            [table_name] => block
        )

    [2] => Array
        (
            [table_schema] => sitebuilder
            [table_name] => layout
        )

    [3] => Array
        (
            [table_schema] => sitebuilder
            [table_name] => menu
        )

    [4] => Array
        (
            [table_schema] => sitebuilder
            [table_name] => page
        )

    [5] => Array
        (
            [table_schema] => sitebuilder
            [table_name] => page_access
        )

    [6] => Array
        (
            [table_schema] => sitebuilder
            [table_name] => site
        )

    [7] => Array
        (
            [table_schema] => sitebuilder
            [table_name] => site_section
        )

    [8] => Array
        (
            [table_schema] => sitebuilder
            [table_name] => sitebuilder_block
        )

    [9] => Array
        (
            [table_schema] => sitebuilder
            [table_name] => sitebuilder_disk_file
        )

    [10] => Array
        (
            [table_schema] => sitebuilder
            [table_name] => sitebuilder_disk_folder
        )

    [11] => Array
        (
            [table_schema] => sitebuilder
            [table_name] => sitebuilder_disk_permission
        )

    [12] => Array
        (
            [table_schema] => sitebuilder
            [table_name] => sitebuilder_disk_settings
        )

    [13] => Array
        (
            [table_schema] => sitebuilder
            [table_name] => sitebuilder_page
        )

    [14] => Array
        (
            [table_schema] => sitebuilder
            [table_name] => sitebuilder_site
        )

    [15] => Array
        (
            [table_schema] => sitebuilder
            [table_name] => sitebuilder_site_disk
        )

    [16] => Array
        (
            [table_schema] => sitebuilder
            [table_name] => sitebuilder_site_user_access
        )

)


COLUMNS:


--- access ---
id : bigint
site_id : bigint
access_code : character varying
role : character varying
created_by : bigint
created_at : timestamp without time zone
updated_by : bigint
updated_at : timestamp without time zone

--- block ---
id : bigint
page_id : bigint
type : character varying
sort : integer
content_json : jsonb
props_json : jsonb
created_by : bigint
created_at : timestamp without time zone
updated_by : bigint
updated_at : timestamp without time zone

--- layout ---
site_id : bigint
settings_json : jsonb
zones_json : jsonb
created_by : bigint
created_at : timestamp without time zone
updated_by : bigint
updated_at : timestamp without time zone

--- menu ---
id : bigint
site_id : bigint
name : character varying
items_json : jsonb
created_by : bigint
created_at : timestamp without time zone
updated_by : bigint
updated_at : timestamp without time zone

--- page ---
id : bigint
site_id : bigint
title : character varying
slug : character varying
parent_id : bigint
sort : integer
status : character varying
published_at : timestamp without time zone
created_by : bigint
created_at : timestamp without time zone
updated_by : bigint
updated_at : timestamp without time zone

--- page_access ---
id : bigint
site_id : bigint
page_id : bigint
access_code : character varying
can_view : boolean
can_edit : boolean
include_children : boolean
created_by : bigint
created_at : timestamp without time zone
updated_at : timestamp without time zone

--- site ---
id : bigint
name : character varying
slug : character varying
home_page_id : bigint
disk_folder_id : bigint
top_menu_id : bigint
settings_json : jsonb
layout_json : jsonb
created_by : bigint
created_at : timestamp without time zone
updated_by : bigint
updated_at : timestamp without time zone
bitrix_group_id : integer
bitrix_group_created_by : integer
bitrix_group_created_at : timestamp without time zone
section_id : bigint

--- site_section ---
id : bigint
name : character varying
sort : integer
created_by : integer
created_at : timestamp without time zone
updated_by : integer
updated_at : timestamp without time zone

--- sitebuilder_block ---
id : bigint
site_id : bigint
page_id : bigint
type : character varying
sort : integer
settings_json : text
is_active : smallint
created_by : bigint
created_at : timestamp without time zone
updated_at : timestamp without time zone

--- sitebuilder_disk_file ---
id : bigint
external_id : bigint
folder_id : bigint
site_id : bigint
block_id : bigint
name : character varying
original_name : character varying
extension : character varying
mime_type : character varying
size : bigint
path : text
hash : character varying
is_deleted : smallint
created_by : bigint
created_at : timestamp without time zone
updated_at : timestamp without time zone
download_url : text
preview_url : text

--- sitebuilder_disk_folder ---
id : bigint
external_id : bigint
parent_id : bigint
site_id : bigint
block_id : bigint
name : character varying
path : text
depth : integer
is_deleted : smallint
created_by : bigint
created_at : timestamp without time zone
updated_at : timestamp without time zone

--- sitebuilder_disk_permission ---
id : bigint
site_id : bigint
block_id : bigint
folder_id : bigint
subject_type : character varying
subject_id : character varying
can_view : smallint
can_upload : smallint
can_create_folder : smallint
can_rename : smallint
can_delete : smallint
can_download : smallint
can_manage_access : smallint
can_edit_settings : smallint
created_at : timestamp without time zone

--- sitebuilder_disk_settings ---
id : bigint
block_id : bigint
site_id : bigint
page_id : bigint
title : character varying
root_folder_id : bigint
view_mode : character varying
allow_upload : smallint
allow_create_folder : smallint
allow_rename : smallint
allow_delete : smallint
allow_download : smallint
show_search : smallint
show_breadcrumbs : smallint
default_sort : character varying
default_sort_direction : character varying
allowed_extensions_json : text
max_file_size : bigint
permission_mode : character varying
use_site_root_fallback : smallint
created_by : bigint
created_at : timestamp without time zone
updated_at : timestamp without time zone

--- sitebuilder_page ---
id : bigint
site_id : bigint
title : character varying
slug : character varying
sort : integer
created_at : timestamp without time zone
updated_at : timestamp without time zone

--- sitebuilder_site ---
id : bigint
name : character varying
code : character varying
root_disk_folder_id : bigint
settings_json : text
created_at : timestamp without time zone
updated_at : timestamp without time zone

--- sitebuilder_site_disk ---
site_id : bigint
root_folder_id : bigint
storage_type : character varying
created_at : timestamp without time zone
updated_at : timestamp without time zone

--- sitebuilder_site_user_access ---
id : bigint
site_id : bigint
user_id : bigint
role_code : character varying
created_at : timestamp without time zone
updated_at : timestamp without time zone
