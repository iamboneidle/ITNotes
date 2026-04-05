___
___
# Tags
#databases
___
# Содержание
- [[#1. Запрос поиска инвалидных пакетов]]
___
# 1. Запрос поиска инвалидных пакетов
```sql
SELECT 
    o.owner,
    o.object_name,
    o.object_type,
    o.status,
    o.created,
    o.last_ddl_time,
    o.timestamp,
    o.edition_name
FROM dba_objects o
WHERE o.status = 'INVALID'
  AND (o.owner IN ('ADMIN_CDB',
                   'AIR',
                   'BIS',
                   'BIS_CRT',
                   'BIS_HBM',
                   'BIS_LIS',
                   'BIS_MBUS',
                   'BIS_OAPI',
                   'BIS_SBMS',
                   'BIS_SCC',
                   'BULK',
                   'CAM',
                   'CCM',
                   'CCM_POM',
                   'CIS',
                   'CIS_MBUS',
                   'CNC',
                   'CRM_CMS',
                   'CSI',
                   'DCS',
                   'EPM',
                   'HEX',
                   'IBR',
                   'OCS',
                   'ORDERS',
                   'PS_MIGR',
                   'SNMP_INT',
                   'SPPBE',
                   'SPPFE',
                   'SPPHAS',
                   'SSO',
                   'UFM_MBUS') 
       OR o.owner LIKE 'GFSHARD_F%' 
       OR o.owner LIKE 'CH20%' 
       OR o.owner LIKE 'USER_CUST_GUARDIAN_00%')
  AND o.object_type IN ('PACKAGE', 'PACKAGE BODY', 'PROCEDURE', 'FUNCTION', 
                       'VIEW', 'TRIGGER', 'TYPE', 'TYPE BODY', 'MATERIALIZED VIEW')
ORDER BY o.owner, o.object_type, o.object_name;
```
___

