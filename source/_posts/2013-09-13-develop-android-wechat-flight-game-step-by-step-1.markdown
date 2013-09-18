---
layout: post
title: "一步一步实现Android打飞机（一）"
date: 2013-09-13 18:31
comments: true
categories: Android
---

<a href="https://github.com/tanqisen/Flight">Android打飞机源码</a>

### 一、概述
  对没有Android开发经验，或对JAVA语言不是太熟悉的同学，希望尝试Android开发应该如何进入呢？为了避免枯燥地看教程、阅读官方sample，学习一大堆不知道什么时候会用上的API，我选择打飞机这个游戏作为切入点。一是因为开发简单游戏并不会涉及过多的平台API以及平台特性，只需要知道基本的贴图、多线程、用户交互等接口就足够了；二是可以把更多的精力放到熟悉语言、培养语感，当然还有游戏本身的逻辑，以及程序设计的通用模式；三是自己动手开发更有趣味性，所以开发中并没有使用游戏框架如cocos2d-x等，仅仅使用了一些Android原始API，毕竟只是为了学习。

### 二、游戏框架
  这是一个典型的贴图游戏，没有复杂的图形变换、动画效果等，你看到的所有效果都是不断移动、替换图片实现的，比如飞机的爆炸效果，就是连续显示几张不同的图片实现的。为了不影响用户交互，比如控制飞机移动，贴图和逻辑控制的工作应该放到一个新的线程中。Android提供了`SurfaceView`类来处理贴图的问题，让`SurfaceView`实现`Runnable`接口并配合`Thread`可以解决在子线程中贴图以及逻辑控制的问题。

<!--more-->

{% codeblock 创建游戏循环 lang:java %}
public class FlightSurfaceView extends SurfaceView implements Callback, Runnable {
  private Thread th = new Thread(this);
  private SurfaceHolder sfh;

  public FlightSurfaceView(Context context) {
    super(context);
    this.setKeepScreenOn(true);
    sfh = this.getHolder();
    sfh.addCallback(this); 

    setFocusable(true);
  }

  public void surfaceCreated(SurfaceHolder holder) {
    // 开始游戏循环
    th.start();
  }

  @Override
  public boolean onTouchEvent(MotionEvent e) { 
    // 主线程事件处理
    return true;
  }

  @Override
  public void run() {
    // 游戏线程，游戏循环
    while (true) {
      try {
        // 游戏逻辑控制
        updateFrame();

        // 渲染当前帧
        renderFrame();

        // 休眠当前线程，线程切换
        Thread.sleep(30);
      } catch (Exception ex) {
      }
    }
  }

  @Override
  public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {
  }

  @Override
  public void surfaceDestroyed(SurfaceHolder holder) {
  }

  private void updateFrame() {
  }

  private void renderFrame() {
  }
}
{% endcodeblock %}

这样，一个基本的游戏循环就建立起来了。但这只是一个什么都不做的空循环，我们还得再这里处理游戏逻辑，以及渲染当前帧。我们先看如何渲染图片、文字。

{% codeblock 绘图过程 lang:java %}
private void renderFrame() {
  Canvas canvas = sfh.lockCanvas(); // 获取并锁定canvas
  canvas.save();    // 保存当前绘图环境

  // TODO 具体绘制

  canvas.restore(); // 恢复先前绘图环境
  sfh.unlockCanvasAndPost(canvas); // 解锁canvas并绘制
}
{% endcodeblock %}

具体绘制时，我们还需要一个绘图工具`Paint`来设置绘制的颜色、字体、样式等参数。

{% codeblock 创建 Paint 绘图及游戏简单控制 lang:java %}
private Paint paint;
private Paint textPaint;
private Bitmap bmp;
private int x;
private int y;

public FlightSurfaceView(Context context) {
  super(context);
  this.setKeepScreenOn(true);
  sfh = this.getHolder();
  sfh.addCallback(this); 

  paint = new Paint();
  paint.setColor(Color.WHITE);   // 设置颜色为白色

  textPaint = new Paint();
  textPaint.setColor(Color.RED); // 设置红色文字画笔

  bmp = BitmapFactory.decodeResource(res, R.drawable.hero);

  x = 100;
  y = 100;

  setFocusable(true);
}

// 简单的游戏控制，这里每帧向右下移动一次图片
private void updateFrame() {
  x ++;
  y ++;

  if (x>320) {
    x = 0;
  }

  if (y>480) {
    y = 0;
  }
}

private void renderFrame() {
  Canvas canvas = sfh.lockCanvas(); // 获取并锁定canvas
  canvas.save();    // 保存当前绘图环境

  // 绘制白色背景
  canvas.drawRect(0,0,320,480,paint);

  // 绘制文字
  canvas.drawText("Flight",0,0,textPaint);

  // 绘制图片
  canvas.drawBitmap(bmp, x, y, paint);
  
  // 绘制文字
  canvas.drawText("Flight",0,0,paint); 

  canvas.restore(); // 恢复先前绘图环境
  sfh.unlockCanvasAndPost(canvas); // 解锁canvas并绘制
}
{% endcodeblock %}

关于Android图形API这里就不详细解释了，后面的内容才是重点。

### 三、游戏资源

游戏中用到的图片、声音等资源可以直接从微信中抠出。这里我使用的是微信iOS版的资源。解开微信安装包，找到了`gameArts-hd.png`、`gameArts-hd.plist`、`gameArts.png`、`gameArts.plist`两套图片资源，`hd`是针对高分辨率的一套图，这里我只用了普通的分辨率`320x480`的资源，一共有两个文件，一个是图片`gameArts.png`，里面包含所有的精灵贴图，以及爆炸效果等，如图所示：

{% img /images/gamearts.png 256 512 gameArts.png %}

> gameArts.png图片加载的是需要注意一点是使用`Bitmap bmp = BitmapFactory.decodeResource(res, R.raw.gamearts);`加载后图片由于Android系统优化，可能改变图片原来的分辨率，导致贴图的时候坐标偏移不对，这里最好使用`InputStream is = this.getResources().openRawResource(R.raw.gamearts); bmp = BitmapFactory.decodeStream(is);`来加载图片

另一个文件是`gameArts.plist`，里面记录了每个精灵在`gameArts.png`中的位置以及尺寸等信息。截取gameArts.plist文件的一个片段，解释一下字段含义：

{% codeblock gameArts.plist 片段 lang:json %}
"hero_fly_1.png":{
  "textureRect":"{ {432, 0}, {66, 82} }",   // 精灵在图片文件中的位置、大小
  "spriteSize":"{66, 82}",
  "spriteColorRect":"{ {1, 1}, {66, 82} }", // 精灵有效像素的偏移及大小
  "spriteTrimmed":true,
  "aliases":[],
  "spriteOffset":"{0, -0}",
  "textureRotated":false, // 是否旋转
  "spriteSourceSize":"{68, 84}"
}
{% endcodeblock %}

这些字段中主要使用了`textureRect`和`spriteColorRect`，由于这套图片中的精灵都没有被旋转，所以`textureRotated`这里不需要。另外由于plist格式在Android中没原生支持，为了解析方便使用的时候将gameArts.plist转成了json格式。

> 可能Android的apk包中的资源跟iOS格式不一样，不过大同小异，只是对文件的解析过程不一样，不影响游戏本身逻辑。可以尝试对不同分辨率载入不同资源。

### 四、 游戏分析

先看游戏中的元素，有随机生产的敌机、玩家控制的英雄飞机、发射的子弹、掉落的降落伞等。

* 英雄：
  1.  正常情况下，玩家可以控制其在屏幕范围内任意移动；
  1.  同时英雄会以一定频率自动发射子弹；
  1.  英雄可以持有不同种类的武器；
  1.  英雄在与敌机碰撞之后会爆炸，爆炸后会闪烁几次，然后游戏结束；
* 敌机：
  1.  敌机有小型飞机、中型飞机、大型飞机；
  1.  小飞机再中弹后即刻爆炸；
  1.  中型飞机和大型飞机在受到一定次数攻击后会有预示将要爆炸的闪烁效果；
  1.  如果再继续被攻击到一定次数，敌机就会爆炸；
  1.  敌机被击爆之后，玩家会获取一定分数；
  1.  敌机的移动不需要玩家控制，而是以一定的随机速度向下移动；
* 子弹：
  1.  只有英雄会发射子弹；
  1.  子弹遇到敌机后会攻击敌机；
  1.  同时子弹本身会消失；
* 降落伞：
  1.  降落伞和英雄碰撞后，降落伞闪烁几次后消失；
  1.  同时玩家可以获得降落伞携带的道具，如双倍子弹、炸弹；

#### 从上面的描述中，我们至少可以抽象出两个基本概率：

1.  类似英雄、敌机、子弹等有生命、有攻击力、可以移动等特性的物体，我们可以抽象成游戏中的`角色`；
1.  游戏角色在不同时刻有不同`状态`，在`不同状态下有不同行为`。比如敌机可以在受到攻击后，可以加快速度冲向英雄，也可以处于闪烁状态，预示飞机即将爆炸；进一步对状态分析后发现，可以抽象出如下几种状态：

    * 正常状态：未受到攻击或只是轻微受伤，角色可以正常移动，攻击（对英雄），冲撞对方等；
    * 被攻击状态：角色受到一定攻击，角色移动速度可能加快，或者带闪烁效果等；
    * 爆炸状态：此时角色已经被消灭，但还未从屏幕中消失，正在爆炸过程中，此时角色已经没有攻击力，也不能移动了；
    * 销毁状态：角色完全爆炸完成会进入这个状态，或角色移动出了屏幕范围，如子弹、敌机移出了屏幕范围也相当于被销毁了，此时角色不会显示在屏幕上了，他的生命周期已经结束；
 
> 角色在不同时刻处于不同的状态，而且状态是可以转化的，如正常飞行的飞机在受到轻微攻击时可能转换到被攻击状态，但受到较大伤害（如玩家投放炸弹）可能使飞机直接转换到爆炸状态；并不是每个角色都存在这四种状态，如小飞机没有被攻击状态，因为小飞机一但被攻击，会直接爆炸。

通过这个分析，我们抽象出了`角色`和`状态`，下一步就是开始大致设计这个游戏模块了。

### 五、模块划分与程序设计

* 首先，游戏资源的加载和管理以及精灵的绘制都与游戏逻辑没直接关系，可以单独设计一个模块；
* 另外角色的创建和回收设计成一个专门的模块，回收角色是为了避免不断的创建新对象，如子弹会源源不断的发射出去，我们不可能每发射一颗子弹就创建一个新对象，这将消耗大量的设备资源，对GC的压力非常大，所以需要回收被销毁的角色，如被消灭的敌机、子弹等，回收后下次需要"创建"新对象时，我们只需要对回收后的角色初始化后再利用。这个模块还可以控制角色的创建时机。例如用一个固定时间间隔创建新的子弹，或者在一个时间段内随机生成一架敌机等；
* 游戏逻辑模块控制角色移动，游戏道具、英雄武器切换、玩家分数、游戏暂停等等；

这个简单的模块划分对接下来的编码实现是非常有帮助的。
