For SaaS applications, each tenant’s data can be stored together in a single database instance and kept isolated from and invisible to other tenants. 

This is efficient in three ways. 

***First***, application improvements apply to all clients. 

***Second***, sharing a database between tenants uses hardware efficiently. 

***Last***, it is much simpler to manage a single database for all tenants than a different database server for each tenant.

Client code requires minimal modifications and can continue to use full SQL capabilities.

Multi-tenant applications have a nice property that we can take advantage of: ***queries usually always request information for one tenant at a time, not a mix of tenants***.


























































