# D3D8 窗口模式垂直同步! Fighting Tearing!


今天BOSS说DEMO有点卡,一看发现是垂直同步的问题!于是乎在HGE的参数上加上了垂直同步的属性,谁知不行,一查资料,发现D3D8 只有在全屏下才能实现垂直同步,万能的微软啊,你不是成心整我们吗?

查了点资料,这种不同步,术语叫:Image Tearing, 大概的意思就是显卡发送到显示器的数据是按行来的,而不是一次传一屏,所以当显卡发送到一个屏的一部分时,这时候放显存里写数据,就会导致图像错位!

原文地址:[www.virtualdub.org/blog/pivot/entry.php](http://www.virtualdub.org/blog/pivot/entry.php?id=74)

解决方法,找了一下午,也没找到用D3D8可以解决的,我还试着使用: GetRasterStatus 来取当前是否已经发送完成,也不行!看来只有换D3D9了!

终于,换成了DX9,冒似好了,结果到了下午,工作了一天,把程序打开一看,居然又开始图像错位了,后来发现,可能的原因是由于在窗口模式下面DX不是独占的,会被GID抢资源,如果程序开多了,而显卡不好的话(我是INTEL集成显卡)可能就会卡,这也只是猜测,还没找到具体原因!

换成DX9后,TEARING问题是解决了,但是又随之而来了新问题,DX9相对于DX8又做了很多改动,原来的程序里面有一套机制,是基于DX8的,将一系列的纹理ALPHA渲染到一个渲染表面中去,然后再将这个渲染表面的纹理渲染到主场景中去.因为HGE使用的DX8,使用D3DXCreateTexture在D3DPOOL\_DEFAULT(显存,DX8 DOC原文:_This is usually video memory, including both local video memory and AGP memory_)创建了一个D3DUSAGE\_RENDERTARGET类型的纹理,因此不能直接锁取出地址来拷贝,所以要先调用CreateImageSurface在D3DPOOL\_SYSTEMMEM(系统内存,_Consumes system RAM but does not reduce pageable RAM_.)创建一个图像表面,调用CopyRects将显存中的表面拷贝到内存中的图像表面中去,然后LockRect锁定内存中的图像表面,将其数据流拷贝到新的纹理中去!

上面是DX8的方法,然而到了DX9,都全了,首先,没有CopyRects了,CreateImageSurface没有了,经过不断的调试,试验,最后使用CreateOffscreenPlainSurface在D3DPOOL\_SYSTEMMEM中创建替代CreateImageSurface,使用GetRenderTargetData将显存中的表面拷贝到内存中的表面去!

这个过程中,发现在DX9中同DX8的CopyRects函数有关的几个函数:

函数                                           源表面位置                        目标表面位置

GetRenderTargetData               POOL\_DEFAULT              D3DPOOL\_SYSTEMMEM

UpdateSurface                           D3DPOOL\_SYSTEMMEM D3DPOOL\_DEFAULT

最后说一下,微软的DX接口老在变,很不爽,可能是不习惯吧!就连DX9的不同版本都在变!
