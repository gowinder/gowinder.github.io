# 关于时间服务的测试结果


经过简单测试，生成500000个计时器，将指针分别插入list和priority\_queue，其中priority\_queue使用vector容器(不能使用list),先使用list自的sort函数，用时516ms,反复测了几次，都在500ms以上550ms以下 ,对priority\_queue进行插入，在500000个的基础上，进行100000次插入，总用时31ms，平均每个0.00031ms  
输出结果如下：  
list sort time:516 ms  
priority\_queue push 100000 event use 31 ms time, average 0.000310 ms per event  
sorted list sort time:516 ms  
sorted list trace 0  
sorted list trace 1  
....  
priority\_queue trace 0  
priority\_queue trace 1  
....  
经过测试，优先队列明显性能高于LIST的自排序，很适合我的要求！
