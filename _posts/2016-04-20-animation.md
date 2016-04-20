---
layout: post
title: Android动画(5.0以下)
date: 2016-04-20 00:00:00 +0800
cover: false
tags: android
subclass: 'post tag-fiction'
categories: 'casper'
cover: 'assets/images/cover6.jpg'
navigation: True
logo: 'assets/images/ghost.png'

---

使用Android两年多了，工作中的动画也动能应付，自认为Android中的动画自己也能用个八九不离十，结果我在学习[Periscope点赞效果](http://www.jianshu.com/p/03fdcfd3ae9c)的时候发现动画的这些高级功能我从没用过、也没见过，静下来仔细想了下，我也并不明白Android动画的实现原理，以及生么时候用什么，从视频以及ApiDemo中看到的LayoutAnimator以及颜色渐变、类似弹簧的反复回弹也都没思路。于是我就研究了下Android的这些动画并记录了下来。


----------


	3.0以前，android支持两种动画模式,tween animation,frame animation，在3.0中又引入了一个新的动画系统：property animation，这三种动画模式在SDK中被称为property animation,view animation,drawable animation
	
# 1. View Animation（Tween Animation）

>View Animation(Tween Animation):补间动画，给出两个关键帧，通过一些算法将给定属性值在给定的时间内在两个关键帧间渐变。(xml方式是在anim文件夹中)

> * View Animation只能用于View对象，而且职能支持一部分功能：位移(translate)、旋转(rotate)、缩放(scale)、透明度渐变(alpha)
> * 还有一个局限性：对于View Animation，它只是改变了View对象绘制的位置，而没有改变View对象本身（例如：做一个位移动画，那么可点击的位置仅仅还是View开始所在的位置，跟移动中的这个动画并无关系）
> * **实现原理**：是父布局不断的画出一个外表一样的图像，不断的通过invalidate 去进行重绘，动画的算法其实都是在Transformation的Matrix矩阵中。
	
# 2. Drawable Animation（Frame Animation）
Drawable Animation（Frame Animation）：帧动画，就像GIF图片，通过一系列Drawable依次显示来模拟动画的效果。(xml方式是在drawable中)

Android中播放GIF图片的时候，可使用这种方式(先分解成单个图片)。

# 3. Property Animation
Android 3.0引入，顾名思义，它是实际更改view的属性，而不像Tween Animation 仅仅是父布局绘制一个替身，所以Property Animation的功能会强大很多。(在包android.animation下)

### 相同：
Property Animation 兼容了 Tween Animation的所有功能：设置动画时间、支持(位移、旋转、缩放、透明度渐变)、类似的监听(开始、结束、取消、重复)、插补器
### 加强功能：
后浪推前浪，后出来的Property Animation带来了更强悍的功能：
#### 1. Evaluators(计算器)：告诉Property Animation系统如何去计算属性值

>- IntEvaluator:用于计算Int类型属性值的计算器。
- FloatEvaluator:用于计算Float类型属性值的计算器。
- ArgbEvaluator:用于计算以16进制形式表示的颜色值的计算器。
- TypeEvaluator:一个计算器接口，它允许你创建你自己的计算器。如果你正在计算一个对象属性并不是int,float或者颜色值类型的，那么你必须实现TypeEvaluator接口去指定如何去计算对象的属性值。

#### 2. 新加了ValueAnimator.AnimatorUpdateListener 监听
　　onAnimationUpdate() - 在动画的每一帧上调用. 在这个方法中，你可以使用ValueAnimator的getAnimatedValue()方法来获取(Evaluators)计算出来的值。
AnimationSet提供了一个把多个动画组合成一个组合的机制，并可设置组中动画的时序关系，如同时播放，顺序播放等。

#### 3. 新增属性动画的同时，也新增了View的属性的设置获取方法
　Example：getLeft、getX、getTranslationX等等 

#### 4. 通过AnimationSet应用多个动画
　　以下例子同时应用5个动画：

    播放anim1；
    同时播放anim2,anim3,anim4；
    播放anim5。
　　
{% highlight java %}
    AnimatorSet bouncer = new AnimatorSet();
    bouncer.play(anim1).before(anim2);
    bouncer.play(anim2).with(anim3);
    bouncer.play(anim2).with(anim4)
    bouncer.play(anim5).after(amin2);
    animatorSet.start();
{% endhighlight %}

#### 5. ObjectAnimator与ValueAnimator之间的关系：
　　其实ObjectAnimator继承与ValueAnimator，ObjectAnimator是为了提供简便的方法，可以直接修改alpha、backgroundColor、translationX、x、y、width等，甚至是一个普通对象的属性，一言以蔽之如果直接通过属性名改属性就用ObjectAnimator

>例子：[Periscope点赞效果实现](http://www.jianshu.com/p/03fdcfd3ae9c)
![这里写图片描述](http://img.blog.csdn.net/20160419161534918)

　　我又换了种方式实现了下，运用我们的属性动画，直接在ViewGroup上画出来：
　　![这里写图片描述](http://img.blog.csdn.net/20160420165151408)

　　[代码传送门](http://download.csdn.net/detail/yuanyang5917/9497167)

#### 6. 同一对象的多个属性同时变化可优化
　　如果需要对一个View的多个属性进行动画可以用ViewPropertyAnimator类，该类对多属性动画进行了优化，会合并一些invalidate()来减少刷新视图，该类在3.1中引入。

　　以下两段代码实现同样的效果：　
{% highlight java %}
PropertyValuesHolder pvhX = PropertyValuesHolder.ofFloat("x", 50f);
PropertyValuesHolder pvhY = PropertyValuesHolder.ofFloat("y", 100f);
ObjectAnimator.ofPropertyValuesHolder(myView, pvhX, pvyY).start();
{% endhighlight %}

#### 7. **原理**：异步根据插补器与Evaluator计算出当前View的属性，再通过handler.post到UI线程，通过反射给View设置当前属性。
[老版本的Property Animation原理讲解](http://zhouyunan2010.iteye.com/blog/1972789)

> ### Tween Animation 与 Property Animation 使用选择：
因为Property Animation最终跟新UI其实也需要重新绘图，所以，属性动画肯定比Tween Animation要更好性能得多，理论上流畅度也稍稍差点。
**所以建议能用Tween Animation的地方，还是使用Tween 动画。**

----------

# 根据ApiDemo罗列了以下几个实用场景（此处会有不少新的用法）：

## 1. 颜色渐变

![这里写图片描述](http://img.blog.csdn.net/20160420095341051)
{% highlight java %}
private static final int RED = 0xffFF8080;
private static final int BLUE = 0xff8080FF;

ValueAnimator colorAnim = ObjectAnimator.ofInt(this, "backgroundColor", RED, BLUE);
    colorAnim.setDuration(3000);
    colorAnim.setEvaluator(new ArgbEvaluator());
    colorAnim.setRepeatCount(ValueAnimator.INFINITE);
    colorAnim.setRepeatMode(ValueAnimator.REVERSE);
    colorAnim.start();
{% endhighlight %}
## 2. 布局显示、不显示、隐藏(LayoutTransition)

![这里写图片描述](http://img.blog.csdn.net/20160420174213674)
ViewGroup中的子元素可以通过setVisibility使其Visible、Invisible或Gone，当有子元素可见性改变时，可以向其应用动画，通过LayoutTransition类应用此类动画：
{% highlight java %}
transition.setAnimator(LayoutTransition.DISAPPEARING, customDisappearingAnim);
{% endhighlight %}
　　通过setAnimator应用动画，第一个参数表示应用的情境，可以以下4种类型：

- APPEARING　　　　　　　　当一个元素变为Visible时对其应用的动画
- CHANGE_APPEARING　　　当一个元素变为Visible时，因系统要重新布局有一些元素需要移动，这些要移动的元素应用的动画
- DISAPPEARING　　　　　　当一个元素变为InVisible时对其应用的动画
- CHANGE_DISAPPEARING　当一个元素变为Gone时，因系统要重新布局有一些元素需要移动，这些要移动的元素应用的动画 disappearing from the 
container.

>步骤：
1. 给view设置LayoutTransition
　 LayoutTransition mTransitioner = new LayoutTransition();
　view.setLayoutTransition(mTransitioner);
2. 设置对应时间
　 Transitioner.setStagger(LayoutTransition.CHANGE_APPEARING, 500);
3. 设置动画
　ObjectAnimator changeIn = ObjectAnimator.ofPropertyValuesHolder(
                        this, pvhLeft, pvhRight, pvhScaleX, pvhScaleY).
                setDuration(mTransitioner.getDuration(LayoutTransition.CHANGE_APPEARING));
        mTransitioner.setAnimator(LayoutTransition.CHANGE_APPEARING, changeIn);
4.  搞定
 
## 3. Keyframes
　 keyFrame是一个 时间/值 对，通过它可以定义一个在特定时间的特定状态，而且在两个keyFrame之间可以定义不同的Interpolator，就相当多个动画的拼接，第一个动画的结束点是第二个动画的开始点。KeyFrame是抽象类，要通过ofInt(),ofFloat(),ofObject()获得适当的KeyFrame，然后通过PropertyValuesHolder.ofKeyframe获得PropertyValuesHolder对象

{% highlight java %}
    Keyframe kf0 = Keyframe.ofFloat(0f, 0f);
    Keyframe kf1 = Keyframe.ofFloat(.9999f, 360f);
    Keyframe kf2 = Keyframe.ofFloat(1f, 0f);
    PropertyValuesHolder pvhRotation =
    PropertyValuesHolder.ofKeyframe("rotation", kf0, kf1, kf2);
    final ObjectAnimator changeOut = ObjectAnimator.ofPropertyValuesHolder(this, pvhLeft, pvhRight, pvhRotation).setDuration(mTransitioner.getDuration(LayoutTransition.CHANGE_DISAPPEARING));
{% endhighlight %}
## 4. 弹跳

![这里写图片描述](http://img.blog.csdn.net/20160420220757849)
弹跳插补器：
{% highlight java %}
bounceAnim.setInterpolator(new BounceInterpolator());
{% endhighlight %}
![这里写图片描述](http://img.blog.csdn.net/20160420220818740)
设置特定一个时间点，显示在那帧位置上
{% highlight java %}
bounceAnim.setCurrentPlayTime(seekTime);
{% endhighlight %}
## 5. 反转切换布局
![这里写图片描述](http://img.blog.csdn.net/20160420221214985)

其实就是一个旋转动画的拼接，一组对立的插补器(加速AccelerateInterpolator、减速DecelerateInterpolator)
{% highlight java %}
ObjectAnimator visToInvis = ObjectAnimator.ofFloat(visibleList, "rotationY", 0f, 90f);
        visToInvis.setDuration(500);
        visToInvis.setInterpolator(accelerator);
        final ObjectAnimator invisToVis = ObjectAnimator.ofFloat(invisibleList, "rotationY",
                -90f, 0f);
        invisToVis.setDuration(500);
        invisToVis.setInterpolator(decelerator);
        visToInvis.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationEnd(Animator anim) {
                visibleList.setVisibility(View.GONE);
                invisToVis.start();
                invisibleList.setVisibility(View.VISIBLE);
            }
        });
        visToInvis.start();
{% endhighlight %}