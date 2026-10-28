# When the Elements of a Priority Queue Are Pointers


Using a priority queue whose elements are pointers, like this:  
typedef priority\_queue<TimeEvent\*, deque <TimeEvent\*>, less<deque<TimeEvent\*>::value\_type>    TIME\_EVENT\_QUEUE;  
Compared this way, the result is a comparison of pointer values. If I want to compare some member of TimeEvent, that doesn't work. After some Googling, there's an approach: replace less with a functor struct:  
struct TimeEventLessComp  
{  
bool operator () (TimeEvent\*& a, TimeEvent\*& b) const  
{  
return a->dwTick > b->dwTick;  
};  
};  
Then change the declaration to:  
typedef priority\_queue<TimeEvent\*, deque <TimeEvent\*>, TimeEventLessComp>    TIME\_EVENT\_QUEUE;

