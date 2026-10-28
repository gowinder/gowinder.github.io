# How to Implement a Time Service


Use a singly or doubly linked list, or a priority queue (priority\_queue).
Each element in the queue is a timer object. A timer object holds the time difference measured from a standard point in time (when the program started running) — the later a timer is supposed to fire, the larger this value — along with the trigger event type.
If you only use a linked list, you have to re-sort after every insertion. With a priority queue the insertion position is handled automatically, so I figure the priority queue should work better, though I still need to test it.
On every heartbeat (the server thread loop), the time service's OnTicker function is called. It checks the time value of the first timer in the queue, and if the current time difference has reached or passed it, the timer is popped out and its trigger event is called.
Right now there are two questions still unanswered:
1. How precise should the time difference be? The best case would be microsecond precision, but then the queue in the time service could grow to hundreds of thousands of entries — how much would that hurt insertion? Needs actual testing.
2. How should the time difference be set? The TICK I'm using now is a DWORD, 32 bits, over 4.2 billion milliseconds, which is 49.71 days in days. If I just store the system TICK plus the time needed until the trigger event, it will very likely overflow. If instead I take the program start time as zero and call the system TICK at that moment SYSTEM\_TICK, then use that as the reference, the time value EVENT\_TICK of a newly added event is:
EVENT\_TICK = NOW\_TICK (current system TICK) - SYSTEM\_TICK + NEED\_TIME (the needed time). In other words there's a conversion step when storing. That way I can store triggers up to 49 days from program start. Haven't quite worked this out yet.

