# 自动生成代码的好处-纪念第一个集成代码生成工作DBCodeGen


突然觉得,一个支持快速开发的引擎,除了有好的底层,方便的模板,还应该具体一套完善的开发工具, 这套工具中,包括了一些自动化生成代码的处理,就是VS里面自动生成MFC应用程序一样.省去咱们很多不必要的工作啊.

自动化生成工具真是谁用谁知道,以前做每个系统时,总有一些重复工作,而这些工作又不方便使用模板或者接口基类来实现,每次复制,粘贴,然后再修改,实在是很烦,突然有一天,一个同仁跟我说,可以用程序来生成这些代码啊.于是呼,我像吃了30年的饭才知道原来可以煮稀饭一样.

之后的过程,那便没有什么新意了,凡是可以自动生成的代码,全部自动生成,定义一个对象的字段,就自动生成数据库表的SQL语句,对应的服务器端的读存此数据对象的类及操作代码,与客户端通讯的消息类,客户端的消息定义,当然,这还没有完,未来还要加上客户端处理的代码等等,凡是可以自动生成的,基本不变的代码,全部自动生成.这样之后,发现写一个程序,再也不用纠结于这些琐碎的工作了,省出更多的时间和精力完成一个系统应该去花脑子去操心的事情.

回头一看,其实自动生成代码从接触编码开始就有了,TC里面的最简单的MAIN函数,VC6的APP生成工具,都是会自动生成一些代码,减少用户操作.现在觉得自己好笨,很没有观察力,明明一直发生在自己身边的事情,却不加以利用.这也许就是高人与凡人之间的区别吧啦.
