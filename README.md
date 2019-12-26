# postgresql-private
Simple validation for tuple modification (insert/update/delete) is by applying constraints. More complex validation can be forced through trigger which is considered expensive by database experts.

The more efficient approach is by implementing validation in user defined function. Regular users are prevented from writing directly to table by issuing INSERT/UPDATE/DELETE. They must invoke provided function to perform those tasks.

But, if you are the function programmer or superuser, the biggest enemy is yourself. You can easily bypass the function, modify table directly and break validation rule you have set.

I have written additional functionality in PostgreSQL version 12.1 version backend to prevent regular user and even superuser from modifying directly table created with "private_modify" option. He or she should call SQL or PLPGSQL function to do that.
~~~
CREATE TABLE public.test (id integer NOT NULL, label text NOT NULL) WITH(private_modify=true);
~~~
Insert into public.test directly and you will have error message.
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
INSERT INTO public.test (id, "label") VALUES ($1, $2);
$BODY$
  LANGUAGE sql VOLATILE SECURITY DEFINER;

CREATE OR REPLACE FUNCTION public.testinsert_plpgsql(i_id integer, t_label text)
  RETURNS void AS
$BODY$
BEGIN
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
But, to allow replication agents such as Slony or Bucardo which set session_replication_role to 'replica' prior table modifications, such restrictions should be relaxed:
~~~
SET session_replication_role TO 'replica';
INSERT INTO public.test VALUES (3, 'ghi');
~~~
## How to apply patch and build PostgreSQL
1. Clone or download patch coming from this repository. Extract as necessary.
2. Download PostgreSQL version 12.1 from https://ftp.postgresql.org/pub/source/v12.1/postgresql-12.1.tar.bz2 or https://ftp.postgresql.org/pub/source/v12.1/postgresql-12.1.tar.gz. Extract to your preferred directory.
3. Apply patch: change directory to the extracted PostgreSQL source code root and run patch -p0 < path-to-patch-file




