# 关于优先队列容器中元素为指针的情况


使用优先队列，元素为指针，如：  
typedef priority\_queue<TimeEvent\*, deque <TimeEvent\*>, less<deque<TimeEvent\*>::value\_type>    TIME\_EVENT\_QUEUE;  
这样比较的话，其结果是比较指针值，如果我要比较TimeEvent中的某个成员，就不行了，经过谷歌，可以采取一个方式，将less替换成一个结构的函数：  
struct TimeEventLessComp  
{  
bool operator () (TimeEvent\*& a, TimeEvent\*& b) const  
{  
return a->dwTick > b->dwTick;  
};  
};  
然后申明时改为：  
typedef priority\_queue<TimeEvent\*, deque <TimeEvent\*>, TimeEventLessComp>    TIME\_EVENT\_QUEUE;
