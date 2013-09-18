---
layout: post
title: "一步一步实现Android打飞机（二）"
date: 2013-09-13 18:31
comments: true
categories: Android
---

<a href="https://github.com/tanqisen/Flight">Android打飞机源码</a>

### 六、角色和状态接口设计

角色基本属性有最大生命值、当前生命值、速度、攻击力、角色位置、角色状态等。角色行为有移动、显示、检测是否与别的角色碰撞、被攻击等。状态属于角色内部属性，可将状态设计成内部类的形式，在特定状态下，角色有特定的行为。例如，正常状态下敌机匀速向屏幕底部移动，受伤后敌机会加速移动，爆炸时敌机原地爆炸、不能移动。

> 注意：我这里说的接口并不是java语言的接口，为了方便实现，使用了普通java类。另外属性权限访问也没有严格限制，还需要改进。

<!--more-->

{% gist 6608198 Art.java %}

详细解释一下：

1.  `ArtFactory`角色工厂，管理角色的创建和回收等；
1.  `GameContext`游戏环境，主要完成贴图的工作，让角色本身不用再关心如何贴图，角色只要设置好当前要绘制的精灵名`sprite`，GameContext自动根据精灵名称、角色位置显示当前角色；
1.  `reset`函数用来初始化角色的所有属性，用于被角色工厂重用；
1.  在`move`和`draw`函数中传入的参数`float deltaTime`表示上一帧到当前帧所用的时间间隔，这个时间用来计算角色当前要偏移的距离(speed * deltaTime)，以及动画持续时间控制，如爆炸效果的一组图片轮播时，每个图片显示0.1秒，就需要用这个参数控制；总之，这个参数目的是消除不同帧率的设备上移动、动画等速度不一致的问题。
1.  这里的碰撞检测只是简单的矩形碰撞，有时看起来不是很真实，特别是飞机的两个角发生碰撞时。如果希望准确的碰撞检测可以在检测到矩形相交后，进一步检测相交区域的像素是否碰撞；

> 碰撞检测时会遇到一个问题是，如果在大分辨率的手机上运行游戏，因为子弹、飞机等的图片很小，如果子弹速度很快，会出现子弹击穿飞机现象。其实这并不是碰撞检测出了问题，而是子弹速度过快导致上一帧子弹距离飞机还有一段距离，但接下来的一帧，由于速度太大导致偏移的距离(speed * deltaTime)直接越过了飞机，显示在了飞机后面，视觉上感觉子弹从飞机身体里穿越过去了。处理这个问题的办法是，子弹的速度不要的过大，同时可以使用使用大点的飞机图片。

> 另外：由于没注意看文档，踩到了Rect的一个坑。Rect.intersects(r1, r2); 和r1.intersect(r2)居然有不一样的行为，你敢信？

### 七、使用Art和ArtState接口实现小型敌机

{% gist 6608198 Enemy1.java %}

可以看出小飞机的实现过程非常简单，只需要设置小飞机的各项属性，以及实现小飞机可能处于的状态类，同时为不同状态实现不同行为就可以了。小飞机没有被攻击被攻击的状态，就不用实现相应状态类。同样地方法可实现英雄、中型飞机、敌机、子弹等角色，这里不再详细列举。

### 八、角色工厂

角色工厂的目标是按一定的时间间隔创建新角色和回收利用被销毁的角色。提供的接口有：
{% codeblock 角色工厂 lang:java %}
// 构造函数
// 基于一定的时间间隔(tick)和随机因素(salt)创建一个cls实例
public ArtFactory(Class<?> cls, float tick, int salt);

// 按时间间隔随机生成新角色，如果生成成功返回实例，失败返回null
public Art genareteArt(GameContext ctx, float deltaTime);

// 重复利用角色a
public void reuseArt(Art a);
{% endcodeblock %}

### 九、武器实现

英雄可以拥有不同武器，武器可以发射子弹。武器我们可以抽象成

{% codeblock 武器接口 lang:java %}
public interface IWeapon {
  // 开火
  public Art fire(float deltaTime);
}
{% endcodeblock %}

实现一个具体武器：

{% codeblock 具体武器实现 lang:java %}
public class Weapon implements IWeapon {
  // 武器持有者
  Art hero = null;

  // 子弹工厂，不同的子弹工厂就可以发射不同的子弹
  ArtFactory bulletFactory = null;
  
  public Weapon(Art hero, ArtFactory bulletFactory) {
    this.hero = hero;
    this.bulletFactory = bulletFactory;
  }
  
  @Override
  public Art fire(float deltaTime) {
    // 发射子弹
    if (bulletFactory != null) {
      Art b = bulletFactory.genareteArt(hero.gameContext, deltaTime);
      if (b != null) {
        // 工厂生成新子弹后，根据武器持有者当前位置，调整发射子弹的初始位置
        b.spriteFrame.offset(hero.spriteFrame.left+(hero.spriteFrame.width() - b.spriteFrame.width())/2, hero.spriteFrame.top-b.spriteFrame.top);
      }
      return b;
    }
    return null;
  }

}
{% endcodeblock %}

### 十、尾声
其他一些基础类如Sprite、SpriteManager、以及GameContext如何绘制精灵、滚动背景如何绘制等细节问题不展开讨论了。基本的组件都创建好了，我们只要实现GameController来使用这些组件、加上一定的游戏逻辑就基本完成了，当然这只是一个游戏的原型，还有很多地方不够完善，这只是对Android开发的初次尝试，希望完成这个游戏的过程中的分析方法、程序设计方法、碰到的问题等可以给大家一些启发。
