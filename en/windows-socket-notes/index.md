# Windows Socket Notes


Error 10055: as soon as the load-test client opened 1000 connections, it threw error 10055. I looked it up and found it meant a lack of resources. Thinking about it afterwards, it must have been connecting too fast — the server couldn't keep up, so the client's buffer filled up. The fix is to add more threads, or pause a little between each connection.  

Later I found that pausing between connections didn't help at all; only adding threads did.

