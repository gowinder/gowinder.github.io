# 文摘:骨骼动画中的坐标插值


摘之:Advanced.Animation.with.DirectX

Here's where using key frames comes in handy. Suppose you want to calculate the orientation of the bones at  
a specific timesay, at 648 milliseconds. That time just so happens to fall between the second and third keys  
(148 milliseconds past the second key). Now, assume that the two transformation matrices represent the  
orientations of each bone in the animation.  
D3DXMATRIX matKey1, matKey2;  
By taking each key and interpolating the values between them, you can come up with a transformation to use  
at any time between the keys. In this example, at 648 milliseconds in the animation, you can interpolate the  
transformations as follows:  
// Get the difference in the matrices  
D3DXMATRIX matTransformation = matKey2 − MatKey1; // 请坐标位移  
// Get keys' times  
float Key1Time = 500.0f;  
float Key2Time = 1200.0f;  
// Get the time difference of the keys  
float TimeDiff = Key2Time − Key1Time; // 算出时间差  
// Get the scalar from animation time and time difference  
float Scalar = (648.0f − Key1Time) / TimeDiff; // 当前帧在两关键帧中间的比值  
// Calculate the interpolated transformation matrix  
matTransformation \*= Scalar; // 算出当前帧的相对变换矩阵  
matTransformation += matKey1; // 算出最终的当前帧的变换矩阵
