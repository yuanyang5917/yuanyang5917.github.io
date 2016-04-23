---
layout: post
title: Object-Oriented
date: 2016-04-23 00:00:00 +0800
cover: false
tags: android
subclass: 'post tag-fiction'
categories: 'casper'
cover: 'assets/images/cover6.jpg'
navigation: True
logo: 'assets/images/ghost.png'

---

> 本篇是根据高焕堂老师的课程归纳记录的，所讲内容包括：Oriented、Class、extends(继承&扩展)、implements、组合、(前三个均有的)Hook方法、IoC(主动调用、被动调用)

## 1. Oriented 面向

- Oriented 意味着一种信仰（万物皆对象）
- Object-oriented 相信任何软件都是由对象所构成的，而且Nothing else.
- 根据上述信仰，电脑语言的设计就简化了，写程序只要定义类别（Class）就行了

## 2. Oriented、Based、Driven、Centered之间的区别?
- Based 的涵义 
例如：Requirement-based基于需求
Requirement-base software development

- Driven 的涵义
例如：Model-driven（模型引导） 、Use Case-driven（用户使用引导）
其实driven 是“引导”，而不是大家常常说说的“驱动”
就像北极星引导我们，指出方向而已，也像汽车司机(Driver)只是引导汽车方向，并没有去驱动汽车；而引擎才是驱动汽车。

- Centered 的涵义
例如：Architecture-centered(建筑中心)   
一切软件开发的活动都围绕着架构，就像圣诞节的糖果和礼物都挂在圣诞树上一样(圣诞树centric)

- Service-Oriented Architecture(SOA)
SOA 是什么含义呢？
相信软件都是以服务构成的，Nothing else!

- 对象(Object)
对个人而言，所认识的东西，皆对象（不认识的就不是对象）
认识的东西，就能说出其特点，并与别的对象比较一番。（其特点包括：对象之特征或属性（Attribute)、对象之行为(Behavior)）

- 软件之对象(Software Object)是由数据(Data)和函数(Function)所组成。

## 3. 类的用途

- 类(Class)是群体(或集合)，而对象是类中的一份子。人们常用［是一个］（is A）来表达对象与类之间的关系

- 例如：
	月亮是一个星球
	毕加索是一个艺术家
	毕加索是一个画家
	张大千是一个画家
- 所以［月球］是对象，属于［星球］类的一个实例（Instance）。毕加索是一个对象，艺术家是一个类，同样的画家也是类，其中画家也是艺术家群体中的小群体（部分集合）。毕加索和张大千同属于［画家］类，所以具有共同点：精于美术绘画

- 类所对应的一群具有同样特征的对象集合。

## 4.  <基类/子类>结构用途 （一）
- 表达继承（继承是扩充的一种）inheritence(继承、遗产)
extends 不仅仅是继承，也是扩充

- 对众多对象加以分门别类，就可以形成一个类继承体系。

- overrite 其实并没有复写，只是名字参数相同，那么调用的时候，会调用子类的这个方法，不过方法内部可以使用super.xxx方法调用父类的此方法。

## 5.  <基类/子类>结构用途（二）表达组合
- 组合与被组合结构上有类似继承的结构（方法相同）
- 组合之间尽量使用接口（针对接口编程，也是一种协议）

## 6.  <基类/子类> 结构的 接口(卡榫函数 Hook)
- (卡榫  字义就是 机械上的起定位和固定作用的卡子 ) Interface 

- 所谓Hook，就是接口，如果两个东西不同时间出现，
则一方会预留虚空，给予另一边于未来时刻能以实体来
填补该空间，两者虚实相依，就密合起来了。设计优良的卡榫，
可以让实体易于新陈代谢、抽换自如（Plug and Play，俗称PnP:即插即用）

- Template Method 模板方法模式（GoF）(模板方法)
![这里写图片描述](http://img.blog.csdn.net/20160423202924230)

- 父类中调用被子类overridable的方法，真是调用的其实是子类的同名同参方法，其实父类的方法依旧是存在的，可以在子类方法中调用super.xxx()实现调用父类此方法。

- 变与不变的分离(Separate code that changes from the code that doesn't)是设计卡榫函数及应用框架之基本原则和手艺。

- 分离出变(Variant)与不变(Invariant)部分之后，就可以将不变部分写在父类别（Super-class)里，而变的部分就写在子类别(Subclass)里。

- 继承与组合都需要Hook Function来对接

## 7. IoC机制与Default函数

- 卡榫函数实现IoC机制

- 控制反转(IoC:Inversion of Control)
IoC机制源于OO语言的继承体系，基类的函数可以主动调用子类的函数，这就是典型的IoC机制。

- 注：子类调用基类：正向
    基类调用子类：控制反转

- 基类与子类之间，控制权是在基类手上，透过Hook函数来调用子类

- 通常基类是写在先，而子类则写在后，这种前辈拥有主导权，进而控制后辈的情形，就统称为控制反转。

- 默认Default行为
基类的重要功能：提供默认(预设)行为
基类可事先定义许多［默认］(Default)函数，这些默认函数可让子类来继承(或调用)之。

## 8. 主动型vs被动型 API
- 模块跟模块之间 API
系统跟人之间   UI : User Interface

- 卡榫函数实现API

- 主动型API：反向/IOC调用
被动型API：正向
(其实是站在父类的角度讲的、基类的视角)

- API的分类
定义(Define)
实作(Implement)
呼叫(Invoke or Call)

- 根据这三个角度，可捋API区分为[主动型]与[被动型]两种。

- 被动型缺乏弹性

- API >= 控制力
接口(interface)是双方接触的地方，也是双方势力或
地盘的界限。谁拥有接口的指定权，谁就掌握控制点，掌握了主动权。

- Activity onCreate 就是一种主动型的API

## 9. 结语&复习：接口与类(别)

- 在OOP里，将接口定义为一种特殊的类别(Class)

- 在C++里，类别包括3种：
1.一般(具象)类别
所有函数都是具象(内有指令)
2.抽象(abstract)类别
有一个或多个函数是抽象的(内无指令)
3.纯粹抽象(pure abstract)类别
所有函数都是抽象的(java 中 的 接口)

- 在UML里，以圆圈来表示接口


