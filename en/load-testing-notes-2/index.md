# Load Testing Notes 2


Yesterday I found that at login the task initialisation was hitting the database far too often. Today, to find the other hot spots for database operations, I added a database-operation counter variable: if the operations get too frequent during login or while running certain calculations, I dump the database operation log and track down the cause.  

I changed the way tasks are initialised, but now the problem is that auto-activation uses WEBSERVICE, and it's far too slow. Not counting the database query time, just calling the web service function alone costs 125MS. Way too slow — I definitely need to build a middle server.

