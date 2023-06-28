> https://severalnines.com/blog/scaling-connections-postgresql-using-connection-pooling/

### Reasons for Connection Pooling
1. PostgreSQL servers are limited to a number of clients they handle at a time ***depending on the memory*** parameter. If this number is surpassed, then you will end up crashing the server.
2. to make a single connection with pipelined requests which can be executed ***simultaneously*** rather than each at a time.
3. connection involves a ***handshake*** that may take 25-35 ms average

For every connection to the database, an **overhead memory between 5 to 10 MB** is used up to cater for a query request.
The core advantage associated with Standalone connection pooling is: minimal overhead cost of about **2kb** for every connection.
