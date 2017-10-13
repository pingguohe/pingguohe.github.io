---

layout: post
title: 天猫物色3D推拉动画
author: 尛破孩-波波

--- 

## 简介
物色中有一个搭配页面，相信许多同学都记忆犹新。前几天Weex同学来问我要其权限，查看这个效果是如何实现的，因此我决定把这个动画的细节和实现告诉大家。这个动画是我刚到天猫花了1天时间完成的，因为其中有非常多的细节需要反复微调打磨。其实，我们在手淘、支付宝等客户端也会发现有类似的动画，但是最大的区别在于手势，猫客这个动画是支持跟手的，这就是我们最需要打磨的地方。为了让大家有一定的认识，先上一个GIF图：

![img](http://upload-images.jianshu.io/upload_images/1603768-3b1d4863fa111132.gif?imageMogr2/auto-orient/strip)

## 从0开始搭建
首先，我们先分析一下这个视图中包含的元素：
![img](http://upload-images.jianshu.io/upload_images/1603768-d95c0199c598aff5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 创建主图区域
+ 主图包含三部分：
	+ UIImageView 用于展示图片
	+ UIView：做一层这招，当附属内容区域上升时，透明度0～0.6
	+ UILabel：对主图的描述文案

+ 实现代码：

	```objC

	- (void)createShowPicView{
		//主图
	_showPicView = [[UIImageView alloc]initWithFrame:CGRectMake(0, 64, 	self.view.bounds.size.width, self.view.bounds.size.height - 110 - 64)];
	_showPicView.userInteractionEnabled = YES;
	_showPicView.backgroundColor = [UIColor whiteColor];
	_showPicView.clipsToBounds = YES;
	_showPicView.contentMode = UIViewContentModeScaleAspectFill;
	_showPicView.image = [UIImage imageNamed:@"putao.jpeg"];
	[self.view addSubview:_showPicView];

	//描述文字
	_descLabel = [[UILabel alloc] init];
	_descLabel.font = [UIFont systemFontOfSize:13.0f];
	_descLabel.numberOfLines = 2;
	_descLabel.backgroundColor = [UIColor clearColor];
	_descLabel.textColor = [UIColor whiteColor];
	_descLabel.shadowColor = [UIColor whiteColor];
	_descLabel.shadowOffset = CGSizeMake(0,1);
	_descLabel.text = @"吃葡萄，不吐葡萄皮";
	[_descLabel sizeToFit];
	CGRect frame = _descLabel.frame;
	frame.origin = CGPointMake(12, self.showPicView.frame.size.height - 20);
	_descLabel.frame = frame;
	[_showPicView addSubview:_descLabel];
	
	//在上面添加遮罩
	_maskView = [[UIView alloc] initWithFrame:self.view.bounds];
	_maskView.backgroundColor = [UIColor blackColor];
	_maskView.userInteractionEnabled = NO;
	_maskView.alpha = 0;
	[self.view addSubview:_maskView];
	}
    ```

#### 创建附属内容区及拖拽区域

+ 拖拽区域其实可以理解为附属内容区域的header，并且添加了Pan手势监控，因此我把这两个区域合二为一来介绍。在布局这部分，我就略过物色中横滑的HorizontalGridView和下面纵划的GridView的介绍（如需这两部分功能的详情，可以跟帖反映，我必分享给大家）。

+ 值得注意的是，我们需要设定一个附属内容区域的最高滑动距离，不能让它无限往上划。这里我设定为恰好让整个附属内容区域全部展示，底部与屏幕底部对齐。
+ 添加手势：为headerView 添加2种手势
	+ UITapGestureRecognizer 支持点击动画，自动升起、回落
	+ UIPanGestureRecognizer 支持拖拽动画，跟手
	
+ 实现代码：

	```objC
	- (void)createShowListView
        {
	_showListView = [[UIView alloc]initWithFrame:CGRectMake(0, self.view.bounds.size.height - 110, self.view.bounds.size.width, self.view.bounds.size.height - 116)];
	_showListView.backgroundColor = [UIColor blackColor];
	_showListView.alpha = 0.8;
	[self.view addSubview:_showListView];

	//限定最高滑动区域
	self.showListMaxDt = self.showListView.frame.origin.y - 116;
	
	UIImageView *headerView = [[UIImageView alloc]initWithFrame:CGRectMake(0, 0, self.showListView.frame.size.width, 26)];
	headerView.tag = 101;
	headerView.contentMode = UIViewContentModeCenter;
	headerView.image = [UIImage imageNamed:@"TMWuse_Beacon"];
	headerView.userInteractionEnabled = YES;
	UITapGestureRecognizer *showClick = [[UITapGestureRecognizer alloc]initWithTarget:self action:@selector(showClick:)];
	[headerView addGestureRecognizer:showClick];
	UIPanGestureRecognizer *beaconPan = [[UIPanGestureRecognizer alloc]initWithTarget:self action:@selector(showListViewPan:)];
	[headerView addGestureRecognizer:beaconPan];
	
	[self.showListView addSubview:headerView];
  }
  ```

#### 点击上升/回落动画
+ 首先我们需要设立一个标志位，用于标志当前附属内容区域的状态，默认当然是未升起的 ```BOOL isShow ＝ No```
+ 附属内容区，只是一个简单的修改Y的坐标值；而在主图区域的遮罩层，这时候就要变化alpha从0到0.6；
+ 重点来了，正视图的ImageView的动画是如何变得呢？我们可以逐段来看，从时间上看，我们分为两个部分；从动画过程中来看，我们可以分为三个部分。那我们就以动画的三部分来看
+ 首先我们需要了解“CATransform3D”的几个属性。我们可以发现，他是一个三维矩阵
	
	CGFloat m11, m12, m13, m14;
  	CGFloat m21, m22, m23, m24;
  	CGFloat m31, m32, m33, m34;
  	CGFloat m41, m42, m43, m44;
  		
  	是不是似曾相识？不错，在大学课本中，我们有这么一个三维变化矩阵的表示
  	
   a  b  c  p 
   d  e  f  q	
   g  h  i   r	
   l  m  n  s 
	
  我们来回忆一下 其中	
  		
   a   b   c 
   d   e   f 
   g   h   i 
 
  产生比例，错切，镜像和旋转等基本变化
  
   l   m   n
 
  
  产生沿x、y、z三轴方向上平移变化
  

   p 
   q 
   r 
 
  产生透视变化
  
   s 

  
  产生等比例缩放变换
  
 然而CATransform3D为我们封装了一些方法来操作，并且是可以在上一个基础效果上做叠加操作
 
 ```CATransform3D CATransform3DTranslate (CATransform3D t, CGFloat tx, CGFloat ty, CGFloat tz)```
t：基础效果，将本次设值叠加于此
tx：X轴偏移位置，往下为正数。
ty：Y轴偏移位置，往右为正数。
tz：Z轴偏移位置，往外为正数。

	```CATransform3D CATransform3DScale (CATransform3D t, CGFloat sx, CGFloat sy, CGFloat sz)```
t：基础效果，将本次设值叠加于此
sx：X轴缩放，代表一个缩放比例，一般都是 0 - 1 之间的数字。
sy：Y轴缩放。
sz：整体比例变换时，也就是m11（sx）== m22（sy）时，若m33（sz）>1，图形整体缩小，若0<1，图形整体放大，若m33（sz）<0，发生关于原点的对称等比变换。
	```CATransform3D CATransform3DRotate (CATransform3D t, CGFloat angle, CGFloat x, CGFloat y, CGFloat z)```
t：基础效果，将本次设值叠加于此
angle：旋转的弧度，所以要把角度转换成弧度：角度 * M_PI / 180。
x：向X轴方向旋转。值范围-1 - 1之间
y：向Y轴方向旋转。值范围-1 - 1之间
z：向Z轴方向旋转。值范围-1 - 1之间
好了，我们用这三个方法之后，就能完成所有动画。

##### 上升
+ 第一部分，以X轴为旋转轴，将图片做旋转
![img](http://upload-images.jianshu.io/upload_images/1603768-b8d8d8e62a35cc8b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

	```objC
	CATransform3D rotationAndPerspectiveTransform = CATransform3DIdentity;
	rotationAndPerspectiveTransform.m34 = 1.0 / 300;
	rotationAndPerspectiveTransform = CATransform3DTranslate(rotationAndPerspectiveTransform, 0, - 	NAVIGATIONBAR_HIGHT * 0.9 , 0);
	rotationAndPerspectiveTransform = CATransform3DScale(rotationAndPerspectiveTransform, 0.9 , 0.9, 1);
	rotationAndPerspectiveTransform = CATransform3DRotate(rotationAndPerspectiveTransform, -12 * M_PI / 180.0f, 1.0f, 	0.0f, 0.0f);
	self.showPicView.layer.transform = rotationAndPerspectiveTransform;
	```	

+ 第二部分 以x轴为旋转轴，将视图倾斜恢复，并且调整视图的Scale，使其缩小。在这两个效果叠加之后，肉眼并非看到顶部回转，而是看到底部被带动到后面，神奇吧
  ![](http://upload-images.jianshu.io/upload_images/1603768-e72fdc23f62a646b.gif?imageMogr2/auto-orient/strip)
  操作的还是那几个方法
  
  ```objC
  CATransform3D rotationAndPerspectiveTransform = CATransform3DIdentity;
	rotationAndPerspectiveTransform.m34 = 1.0 / 300;
	rotationAndPerspectiveTransform = CATransform3DTranslate(rotationAndPerspectiveTransform, 0,  - NAVIGATIONBAR_HIGHT * 1.8 , 0);
	rotationAndPerspectiveTransform = CATransform3DScale(rotationAndPerspectiveTransform, 0.8, 0.8, 1);
	rotationAndPerspectiveTransform = CATransform3DRotate(rotationAndPerspectiveTransform, 0.0f, 1.0f, 0.0f, 0.0f);
	self.showPicView.layer.transform = rotationAndPerspectiveTransform;
  ```
+ 第三部分 由于我们一直操作的是layer，对frame并没有改变，这使得我们在之后的操作中会带来许多的麻烦，所以最后动画停止之后，我们需要设置frame，并将Scale变为1

	```objC
	float top = self.showPicView.frame.origin.y;
	float left = self.showPicView.frame.origin.x;
	CALayer *layer = self.showPicView.layer;
	CATransform3D rotationAndPerspectiveTransform = CATransform3DIdentity;
	rotationAndPerspectiveTransform = CATransform3DScale(rotationAndPerspectiveTransform, 1, 1, 1);
	self.showPicView.layer.transform = rotationAndPerspectiveTransform;
	self.showPicView.transform = CGAffineTransformMakeScale(0.8,0.8);
	self.showPicView.frame = CGRectMake(left, top, self.showPicView.frame.size.width, self.showPicView.frame.size.height);
	```
##### 回落
+ 回落与上升刚好相反，这里不细说了，直接上代码，补充完之后，我们就能通过点击来完成上升和回落的效果了

	```objC
	[self.navigationController setNavigationBarHidden:NO animated:YES];
	[UIView animateWithDuration:0.5 animations:^{
			self.showListView.frame = CGRectMake(0, self.view.bounds.size.height - 110, self.view.bounds.size.width, self.view.bounds.size.height - 116);
			self.maskView.alpha = 0;
		}];
		
		//回复原状
		self.showPicView.transform = CGAffineTransformMakeScale(1,1);
		self.showPicView.frame = CGRectMake(0, NAVIGATIONBAR_HIGHT + STATEBAR_HIGHT, self.showPicView.frame.size.width, self.showPicView.frame.size.height);
		self.narrow = NO;
		CATransform3D rotationAndPerspectiveTransform = CATransform3DIdentity;
		rotationAndPerspectiveTransform.m34 = 1.0 / 300;
		rotationAndPerspectiveTransform = CATransform3DTranslate(rotationAndPerspectiveTransform, 0, - NAVIGATIONBAR_HIGHT * 1.8 , 0);
		rotationAndPerspectiveTransform = CATransform3DScale(rotationAndPerspectiveTransform, 0.8, 0.8, 1);
		rotationAndPerspectiveTransform = CATransform3DRotate(rotationAndPerspectiveTransform, 0.0f, 1.0f, 0.0f, 0.0f);
		self.showPicView.layer.transform = rotationAndPerspectiveTransform;
		
		
		[UIView animateWithDuration:0.25 animations:^{
			CALayer *layer = self.showPicView.layer;
			layer.zPosition = -200;
			CATransform3D rotationAndPerspectiveTransform = CATransform3DIdentity;
			rotationAndPerspectiveTransform.m34 = 1.0 / 300;
			rotationAndPerspectiveTransform = CATransform3DTranslate(rotationAndPerspectiveTransform, 0,  - NAVIGATIONBAR_HIGHT * 0.9 , 0);
			rotationAndPerspectiveTransform = CATransform3DScale(rotationAndPerspectiveTransform, 0.9 , 0.9, 1);
			rotationAndPerspectiveTransform = CATransform3DRotate(rotationAndPerspectiveTransform, -12 * M_PI / 180.0f, 1.0f, 0.0f, 0.0f);
			self.showPicView.layer.transform = rotationAndPerspectiveTransform;
			
		} completion:^(BOOL finished) {
			[UIView animateWithDuration:0.25 animations:^{
				CALayer *layer = self.showPicView.layer;
				layer.zPosition = -200;
				CATransform3D rotationAndPerspectiveTransform = CATransform3DIdentity;
				rotationAndPerspectiveTransform.m34 = 1.0 / 300;
				rotationAndPerspectiveTransform = CATransform3DTranslate(rotationAndPerspectiveTransform, 0,  0 , 0);
				rotationAndPerspectiveTransform = CATransform3DScale(rotationAndPerspectiveTransform, 1.0, 1.0, 1);
				rotationAndPerspectiveTransform = CATransform3DRotate(rotationAndPerspectiveTransform, 0.0f, 1.0f, 0.0f, 0.0f);
				self.showPicView.layer.transform = rotationAndPerspectiveTransform;
			}];
		}];
	```
![](http://upload-images.jianshu.io/upload_images/1603768-ad0c6d451eb7a19b.gif?imageMogr2/auto-orient/strip)

#### 跟手拖拽动画
+ 拖拽的3个时期
	+ UIGestureRecognizerStateBegan时期
	
		+ 获取滑动触碰的开始点：
		``` startPoint_Y = [recognizer locationInView:self.view.window].y;```
		+ 获取当前附属View的Y值：
		```viewPoint_Y = self.showListView.frame.origin.y;```
		
	+ UIGestureRecognizerStateChanged时期
		+ 获取当前的触碰点：
		```changePoint_Y = [recognizer locationInView:self.view.window].y;```
		+ 获取附属View的Y点：
		```float move_Y = viewPoint_Y + (changePoint_Y - startPoint_Y);```
		+ 我们需要需设置最高点和最低点，免得过度滑动：
		```
		if (move_Y > self.view.bounds.size.height - 110)
		{
			move_Y = self.view.bounds.size.height - 110;
		}
		else if(move_Y < 116)
		{
			move_Y = 116;
		}
		```
		+ 设置附属View的Y值，并且设置此次滑动占整个动画中的进度，每个进度将会对应主图的动画进度
		```
		self.showListView.frame = CGRectMake(self.showListView.frame.origin.x, move_Y, self.showListView.frame.size.width, self.showListView.frame.size.height);
			[self showPicViewChangeProgress:((self.view.bounds.size.height - 116) - self.showListView.frame.origin.y)/self.showListMaxDt];
			[recognizer setTranslation:CGPointZero inView:self.view.window];
		```
	+ UIGestureRecognizerStateEnded时期
		+ 将当前进度传给```-(void)showPicViewAnimationProgress:```方法，补全动画

		
+ 	showPicViewChangeProgress方法
	+ 之前我们动画按照时间上区分是2个阶段，那么，跟手动画我们也可以分为前半部分和后半部分
	+ 前半部分相当于点击动画的第一部分，还是操作那3个函数，只是在函数的设置当中带入了当前的进度，进度值为0-1
	
		```objC
		if (self.navigationController.navigationBarHidden)
		{
			[self.navigationController setNavigationBarHidden:NO animated:YES];
		}
		CALayer *layer = self.showPicView.layer;
		layer.zPosition = -200;
		CATransform3D rotationAndPerspectiveTransform = CATransform3DIdentity;
		rotationAndPerspectiveTransform.m34 = 1.0 / 300;
		rotationAndPerspectiveTransform = CATransform3DTranslate(rotationAndPerspectiveTransform, 0,  - NAVIGATIONBAR_HIGHT * progress * 1.8 , 0);
		rotationAndPerspectiveTransform = CATransform3DScale(rotationAndPerspectiveTransform, 1.0 - (0.2 * progress) , 1.0 - (0.2 * progress), 1);
		rotationAndPerspectiveTransform = CATransform3DRotate(rotationAndPerspectiveTransform, -(24 * progress)*M_PI / 180.0f, 1.0f, 0.0f, 0.0f);
		self.showPicView.layer.transform = rotationAndPerspectiveTransform;
		```
	+ 后半部分相当于动画的第二部分，各个函数也带上进度值：
	
		```objC
		if (!self.navigationController.navigationBarHidden)
		{
			[self.navigationController setNavigationBarHidden:YES animated:YES];
		}
		CALayer *layer = self.showPicView.layer;
		layer.zPosition = -200;
		CATransform3D rotationAndPerspectiveTransform = CATransform3DIdentity;
		rotationAndPerspectiveTransform.m34 = 1.0 / 300;
		rotationAndPerspectiveTransform = 	CATransform3DTranslate(rotationAndPerspectiveTransform, 0,  - NAVIGATIONBAR_HIGHT * progress * 1.8 , 0);
		rotationAndPerspectiveTransform = 	CATransform3DScale(rotationAndPerspectiveTransform, 1.0 - (0.2 * progress), 1.0 - (0.2 * progress), 1);
		rotationAndPerspectiveTransform = CATransform3DRotate(rotationAndPerspectiveTransform, -(12 - 24*(progress - 0.5))*M_PI / 180.0f, 1.0f, 0.0f, 0.0f);
		self.showPicView.layer.transform = rotationAndPerspectiveTransform;
		```
		

+ 	showPicViewAnimationProgress方法
	+ 此方法做手指离开屏幕后补全，如果进度>0.5则继续<0.5则返回滑动前的状态
	+ 待动画结束之后，要做的正式点击动画中第三部分需要做的事情，至此，跟手动画与点击动画一一对应，动画效果也一模一样
	
		```objC

		if(progress <= 0.5)
		{
		[UIView animateWithDuration:0.25 animations:^{
			CALayer *layer = self.showPicView.layer;
			layer.zPosition = -200;
			CATransform3D rotationAndPerspectiveTransform = CATransform3DIdentity;
			rotationAndPerspectiveTransform.m34 = 1.0 / 300;
			rotationAndPerspectiveTransform = CATransform3DTranslate(rotationAndPerspectiveTransform, 0, 0, 0);
			rotationAndPerspectiveTransform = CATransform3DScale(rotationAndPerspectiveTransform, 1.0, 1.0, 1);
			rotationAndPerspectiveTransform = CATransform3DRotate(rotationAndPerspectiveTransform, 0.0f, 1.0f, 0.0f, 0.0f);
			self.showPicView.layer.transform = rotationAndPerspectiveTransform;
			self.showListView.frame = CGRectMake(0, self.view.bounds.size.height - 110, self.view.bounds.size.width, self.view.bounds.size.height - 116);
			self.maskView.alpha = 0;
		}completion:^(BOOL finished) {
			self.narrow = NO;
			self.isShow = NO;
		}];
      }
      else
      {
		//已经缩小，说明已经达到固定位置了，不需要再做动画
		if (self.narrow)
		{
			return;
		}
		[UIView animateWithDuration:0.25 animations:^{
			CALayer *layer = self.showPicView.layer;
			layer.zPosition = -200;
			CATransform3D rotationAndPerspectiveTransform = CATransform3DIdentity;
			rotationAndPerspectiveTransform.m34 = 1.0 / 300;
			rotationAndPerspectiveTransform = CATransform3DTranslate(rotationAndPerspectiveTransform, 0,  - NAVIGATIONBAR_HIGHT * 1.8 , 0);
			rotationAndPerspectiveTransform = CATransform3DScale(rotationAndPerspectiveTransform, 0.8 , 0.8, 1);
			rotationAndPerspectiveTransform = CATransform3DRotate(rotationAndPerspectiveTransform, 0.0f, 1.0f, 0.0f, 0.0f);
			self.showPicView.layer.transform = rotationAndPerspectiveTransform;
			self.showListView.frame = CGRectMake(0, 116, self.showListView.frame.size.width, self.showListView.frame.size.height);
			self.maskView.alpha = 0.6;
		} completion:^(BOOL finished) {
			//改变原状
			float top = self.showPicView.frame.origin.y;
			float left = self.showPicView.frame.origin.x;
			CALayer *layer = self.showPicView.layer;
			layer.zPosition = -200;
			CATransform3D rotationAndPerspectiveTransform = CATransform3DIdentity;
			rotationAndPerspectiveTransform = CATransform3DScale(rotationAndPerspectiveTransform, 1, 1, 1);
			self.showPicView.layer.transform = rotationAndPerspectiveTransform;
			self.showPicView.transform = CGAffineTransformMakeScale(0.8,0.8);
			self.showPicView.frame = CGRectMake(left, top, self.showPicView.frame.size.width, self.showPicView.frame.size.height);
			self.narrow = YES;
			self.isShow = YES;
		}];
		}
       ```
## 结束
好了，天猫物色搭配页面的推拉效果我们已经完成了，细节的打磨是一个与视觉和交互沟通指定的过程，各位同学需要耐心调整，最后在附件中附上本文中的Demo