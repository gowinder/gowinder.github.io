# D3D8 Windowed Mode Vsync! Fighting Tearing!


Today the boss said the DEMO felt a bit choppy. One look and it turned out to be a vertical sync problem! So I added the vsync attribute to HGE's parameters — no dice. A bit of reading and I found that D3D8 can only do vertical sync in fullscreen mode. Almighty Microsoft, are you doing this to us on purpose?

I dug up some references. This lack of sync is called Image Tearing. Roughly: the data the graphics card sends to the monitor goes row by row, not a whole screen at a time, so when the card has sent part of a screen and you write data into video memory at that moment, the image ends up misaligned!

Original article: [www.virtualdub.org/blog/pivot/entry.php](http://www.virtualdub.org/blog/pivot/entry.php?id=74)

As for a fix, I searched all afternoon and found nothing that works with D3D8. I even tried using GetRasterStatus to check whether the current frame had finished being sent — that didn't work either! Looks like the only option is to move to D3D9!

Eventually I switched to DX9 and it seemed fine. Then in the afternoon, after a full day of work, I opened the program and the image was misaligned again. Later I worked out the likely cause: in windowed mode DX isn't exclusive, so the GID can grab resources. If you have a lot of programs open and a weak graphics card (mine is Intel integrated graphics), it can stutter. That's only a guess though — I haven't pinned down the actual cause yet!

After switching to DX9 the TEARING problem was solved, but a new problem came with it. DX9 made a lot of changes compared with DX8, and the old program had a mechanism built on DX8: render a series of alpha textures into a render surface, then render that surface's texture into the main scene. Because HGE uses DX8, D3DXCreateTexture created a texture of type D3DUSAGE\_RENDERTARGET in D3DPOOL\_DEFAULT (video memory; the DX8 DOC says: _This is usually video memory, including both local video memory and AGP memory_), so you can't lock it directly to get an address and copy. You first had to call CreateImageSurface to create an image surface in D3DPOOL\_SYSTEMMEM (system memory, _Consumes system RAM but does not reduce pageable RAM_.), call CopyRects to copy the surface from video memory into the image surface in memory, then LockRect the in-memory image surface and stream its data into the new texture!

That's the DX8 way. In DX9 it's all different: first, CopyRects is gone, CreateImageSurface is gone. After a lot of debugging and experimenting, I ended up using CreateOffscreenPlainSurface in D3DPOOL\_SYSTEMMEM to replace CreateImageSurface, and GetRenderTargetData to copy the surface from video memory into the in-memory surface!

Along the way I found the DX9 functions that correspond to DX8's CopyRects:

Function                                           Source surface pool                    Destination surface pool

GetRenderTargetData               POOL\_DEFAULT              D3DPOOL\_SYSTEMMEM

UpdateSurface                           D3DPOOL\_SYSTEMMEM D3DPOOL\_DEFAULT

One last thing: Microsoft's DX interfaces keep changing, which is really annoying — maybe I'm just not used to it! Even different versions of DX9 keep changing!

