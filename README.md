# postgresql-private
Simple validation for tuple modification (insert/update/delete) is by applying constraints. More complex validation can be forced through trigger which is considered expensive by database experts.

The more efficient approach is by implementing validation in user defined function. Superuser prevents regular users from issuing INSERT/UPDATE/DELETE directly to table by revoking their privileges. They must invoke provided function to perform those tasks.

But, if you are the function programmer or superuser, the biggest enemy is yourself. You can easily bypass the function, modify table directly and break validation rule that you have set.

I have written additional functionality in PostgreSQL version 12.1 version backend to prevent regular user and even superuser from modifying directly table created with "private_modify" option. He or she should call SQL, PLPGSQL or other SPI-based function function to do that.
~~~
CREATE TABLE public.regular (id integer NOT NULL, label text NOT NULL);
CREATE TABLE public.test (id integer NOT NULL, label text NOT NULL) WITH(private_modify=true);
~~~
Check table options in system table:
~~~
SELECT n.nspname, relname, reloptions
FROM pg_class c 
INNER JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relname in('regular','test') AND n.nspname='public';

 nspname | relname |      reloptions
---------+---------+-----------------------
 public  | regular | 
 public  | test    | {private_modify=true}
~~~
Insert into public.regular directly and it works as usual.
~~~
INSERT INTO public.regular VALUES (1, 'abc');
~~~
Now, insert into public.test directly and you will have error message.
~~~
INSERT INTO public.test VALUES (1, 'abc');
ERROR:  do not modify table with "private modify" option outside SQL or PLPGSQL function
~~~
Update or delete will have the same error message.

So let us create function in SQL language and PLPGSQL language:
~~~
CREATE OR REPLACE FUNCTION public.testinsert_sql(i_id integer, t_label text)
  RETURNS void AS
$BODY$
/*do necessary validation*/
INSERT INTO public.test (id, "label") VALUES ($1, $2);
$BODY$
  LANGUAGE sql VOLATILE SECURITY DEFINER;

CREATE OR REPLACE FUNCTION public.testinsert_plpgsql(i_id integer, t_label text)
  RETURNS void AS
$BODY$
BEGIN
/*do necessary validation*/
	INSERT INTO public.test (id, "label") VALUES ($1, $2);
	RETURN;
END;$BODY$
  LANGUAGE plpgsql VOLATILE SECURITY DEFINER;
~~~
And use those functions for table modifications without error:
~~~
SELECT public.testinsert_sql(1, 'abc');
SELECT public.testinsert_plpgsql(2, 'def');
~~~
Check the result:
~~~
SELECT * FROM public.test;
 id | label 
----+-------
  1 | abc
  2 | def
~~~
But, to allow replication agents such as Slony or Bucardo which set session_replication_role to 'replica' prior to table modifications, such restrictions should be relaxed:
~~~
SET session_replication_role TO 'replica';
INSERT INTO public.test VALUES (3, 'ghi');
~~~
## How to apply patch and build PostgreSQL
1. Clone or download patch from https://github.com/AbdulYadi/postgresql-private. Extract as necessary.
2. Download PostgreSQL version 12.1 from https://ftp.postgresql.org/pub/source/v12.1/postgresql-12.1.tar.bz2 or https://ftp.postgresql.org/pub/source/v12.1/postgresql-12.1.tar.gz. Extract to your preferred directory.
3. Apply patch: change directory to the extracted PostgreSQL source code root and run patch -p0 < path-to-downloaded-patch-file.
4. Build as instructed in PostgreSQL's INSTALL file.
## Warning
This patch is still an experiment so do not put into production server.



