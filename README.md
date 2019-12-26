# postgresql-private
Simple validation for tuple modification (insert/update/delete) is by applying constraints. More complex validation can be forced through trigger which is considered expensive by database experts.

The more efficient approach is by implementing validation in user defined function. Regular users are prevented from writing directly to table by issuing INSERT/UPDATE/DELETE. They must invoke provided function to perform those tasks.

But, if you are the function programmer or superuser, the biggest enemy is yourself. You can easily bypass the function, modify table directly and break validation rule you have set.

I have written additional functionality in PostgreSQL version 12.1 version to prevent regular user and even superuser from modifying directly table created with "private_modify" option. He or she should call SQL or PLPGSQL function to do that.
~~~
CREATE TABLE public.test (id integer NOT NULL, label text NOT NULL) WITH(private_modify=true);
~~~
Insert into public.test directly:
~~~
CREATE TABLE public.test (id integer NOT NULL, label text NOT NULL) WITH(private_modify=true);
ERROR:  do not modify table with "private modify" option outside SQL or PLPGSQL function
~~~