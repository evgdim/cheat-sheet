# Data pump
**check DATA_PUMP_DIR**

```select * from dba_directories where DIRECTORY_NAME = 'DATA_PUMP_DIR';```

```create or replace directory data_pump_dir as 'D:\orabcp';```

```grant read,write on directory DATA_PUMP_DIR to <user>;```

**export**

```expdp c##test/test@ORCL schemas=c##test directory=DATA_PUMP_DIR dumpfile=testdump.dmp logfile=expdpTest.log```

**import**

```impdp c##test/test@ORCL schemas=c##test directory=DATA_PUMP_DIR dumpfile=testdump.dmp logfile=impdpTest.log```

#Clear schema
```select 'drop '||object_type||' '|| object_name||';' from user_objects;```

#Regex split
```
  with test as 
  (
    select 'ABC;DEF;GHI;JKL;MNO' str from dual  
  )  
  select regexp_substr (str, '[^;]+', 1, rownum) split  
  from test  
  connect by level <= length (regexp_replace (str, '[^;]+'))  + 1;
```
# Create tablespace and user/schema
```
CREATE BIGFILE TABLESPACE myuser_ts
  DATAFILE 'myuser_ts.dat'
  SIZE 20M AUTOEXTEND ON;

alter session set "_ORACLE_SCRIPT"=true; -- only for 12c if you want to create common user without c## prefix

CREATE USER myuser IDENTIFIED BY myuser
DEFAULT TABLESPACE myuser_ts;

GRANT CONNECT TO myuser;
GRANT RESOURCE TO myuser;

ALTER USER myuser QUOTA UNLIMITED ON myuser_ts;

--revert
drop user myuser cascade;
drop tablespace myuser_ts including contents and datafiles;
```
# Generate Java class and Spring row mapper from trables

```
DECLARE
  TARGET_SCHEMA VARCHAR2(20) := 'MYUSER';
  NL char := chr(10);
  CLASS_STR_TMP VARCHAR2(32767) := '@Data'||NL||'public class ';-- lombok @Data
  ROWMAPPER_STR_TMP VARCHAR2(32767) := 'public class ';
  JAVA_DATA_TYPE VARCHAR2(100);
  JAVA_CLASS_NAME VARCHAR2(100);
  JAVA_PROP_NAME VARCHAR2(100);
BEGIN
   FOR t IN (SELECT * FROM ALL_TABLES WHERE owner = TARGET_SCHEMA) LOOP
    JAVA_CLASS_NAME := replace(initcap(t.table_name),'_');
    CLASS_STR_TMP := CLASS_STR_TMP || JAVA_CLASS_NAME || ' {'|| NL;
    ROWMAPPER_STR_TMP := ROWMAPPER_STR_TMP || JAVA_CLASS_NAME || 'RowMapper implements RowMapper<' || JAVA_CLASS_NAME ||'> {' || NL ||
      '   @Override' || NL ||
      '   public ' || JAVA_CLASS_NAME || ' mapRow(ResultSet rs, int i) throws SQLException { ' || NL ||
      '      ' || JAVA_CLASS_NAME || ' ' || lower(JAVA_CLASS_NAME) || ' = new ' || JAVA_CLASS_NAME || '();' || NL;
    
    FOR c IN (SELECT * FROM ALL_TAB_COLS WHERE TABLE_NAME = t.table_name ORDER BY COLUMN_ID) LOOP
      JAVA_DATA_TYPE := CASE c.DATA_TYPE
        WHEN 'VARCHAR2' THEN 'String'
        WHEN 'NUMBER' THEN 'BigDecimal'
        WHEN 'INT' THEN 'Integer'
        WHEN 'DATE' THEN 'Timestamp'
        WHEN 'TIMESTAMP' THEN 'Timestamp'
        ELSE '<Unknown>'
      END;
      JAVA_PROP_NAME := replace(initcap(c.COLUMN_NAME),'_');
      CLASS_STR_TMP := CLASS_STR_TMP || '   private ' || JAVA_DATA_TYPE || ' ' || JAVA_PROP_NAME || ';' || NL;
      ROWMAPPER_STR_TMP := ROWMAPPER_STR_TMP || '      ' ||
        lower(JAVA_CLASS_NAME) || '.set' || initcap(JAVA_PROP_NAME) || '(rs.get' || JAVA_DATA_TYPE || '("'||c.COLUMN_NAME||'"));' || NL;
    END LOOP;
    CLASS_STR_TMP := CLASS_STR_TMP || NL || '}';
    ROWMAPPER_STR_TMP := ROWMAPPER_STR_TMP || '      return ' || lower(JAVA_CLASS_NAME) || ';' || NL ||
      '   }' || NL ||
    '}';
    
     dbms_output.put_line('===================================');
     dbms_output.put_line(CLASS_STR_TMP);
     dbms_output.put_line('');
     dbms_output.put_line('...................................');
     dbms_output.put_line('');
     dbms_output.put_line(ROWMAPPER_STR_TMP);
     dbms_output.put_line('===================================');
     CLASS_STR_TMP := '';
     ROWMAPPER_STR_TMP := '';
   END LOOP;  
EXCEPTION
   WHEN others THEN
      dbms_output.put_line('asd');
END;
```
# Find all FK to a table
```
select table_name, constraint_name, status, owner
from all_constraints
where r_owner = :r_owner
and constraint_type = 'R'
and r_constraint_name in
 (
   select constraint_name from all_constraints
   where constraint_type in ('P', 'U')
   and table_name = :r_table_name
   and owner = :r_owner
 )
order by table_name, constraint_name
```

# Find unindexed FKs
```
select
case
   when b.table_name is null then
      'unindexed'
   else
      'indexed'
end as status,
   a.table_name      as table_name,
   a.constraint_name as fk_name,
  a.fk_columns      as fk_columns,
  b.index_name      as index_name,
  b.index_columns   as index_columns
from
(
   select 
    a.table_name,
   a.constraint_name,
   listagg(a.column_name, ',') within
group (order by a.position) fk_columns
from
   dba_cons_columns a,
   dba_constraints b
where
   a.constraint_name = b.constraint_name
and 
   b.constraint_type = 'R'
and 
   a.owner = '&&schema_owner'
and 
   a.owner = b.owner
group by 
   a.table_name, 
   a.constraint_name
) a
,(
select 
   table_name,
   index_name,
   listagg(c.column_name, ',') within
group (order by c.column_position) index_columns
from
   dba_ind_columns c
where 
   c.index_owner = '&&schema_owner'
group by
   table_name, 
   index_name
) b
where
   a.table_name = b.table_name(+)
and 
   b.index_columns(+) like a.fk_columns || '%'
order by 
   1 desc, 2;
```
