# Test Results for the Time Service


After a simple test: generate 500,000 timers, then insert the pointers into a list and into a priority\_queue respectively, with the priority\_queue using a vector container (it can't use list). First I used list's own sort function: 516ms, and I ran it several times, always above 500ms and below 550ms. For the priority\_queue, on top of the existing 500,000 I did 100,000 insertions, 31ms in total, 0.00031ms per event on average.  
The output looks like this:  
list sort time:516 ms  
priority\_queue push 100000 event use 31 ms time, average 0.000310 ms per event  
sorted list sort time:516 ms  
sorted list trace 0  
sorted list trace 1  
....  
priority\_queue trace 0  
priority\_queue trace 1  
....  
So the test shows the priority queue is clearly faster than sorting the LIST myself — a great fit for what I need!

