# Stress Testing Notes 1


Closed beta is almost here, so we've started stress testing.  
1000 connections was fine, but the problems showed up after login: handling a single login takes one second, so you can imagine how long 1000 of them would take.  
That obviously won't do.  
So I added some test statements to find out who was dragging things out.  
In the end it turned out to be the player task initialisation at login — there are over 900 tasks, each with sub-tasks, and every sub-task does a SELECT against the database whether or not there's a record. No wonder. Tomorrow I'll get someone to fix it: completed tasks and tasks that haven't started yet don't need to be read at all. That should speed things up a lot.  
Then I found another problem with log writing. In the function called on every TICK, if a CHECK-style log fires an error, it keeps writing and the server stops responding. Apart from turning off logging to disk, there's almost nothing else you can do. When the UI thread displays things, each time it pulls a log message it now checks whether that would exceed the time available for this TICK; if it would, the unprocessed TICKs get handled next time, so the UI doesn't freeze.

