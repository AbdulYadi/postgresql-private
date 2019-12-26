# postgresql-private
Simple validation for tuple modification (insert/update/delete) is by applying constraints. More complex validation can be forced through trigger which is considered expensive by database experts.

The more efficient approach is by implementing validation in user defined function. Regular users are prevented from writing directly to table by issuing INSERT/UPDATE/DELETE. They must invoke provided function to perform those tasks.

But, if you are the function programmer or superuser, the biggest enemy is yourself. You can easily bypass the function, modify table directly and break validation rule you have set.

