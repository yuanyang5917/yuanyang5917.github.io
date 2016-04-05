---
layout: post
title: "注解初始化控件(ButterKnife方式)<上>"
date: 2016-04-05 00:00:00 +0800
cover: false
tags: android
subclass: 'post tag-fiction'
categories: 'casper'
cover: 'assets/images/cover6.jpg'
navigation: True
logo: 'assets/images/ghost.png'

---


>在学习了XUtils的注入方式之后，看了下ButterKnife的实现方式，结果发现完全不一样，然后借鉴网上的博客，结果发现用的都是Eclipse以及旧版本的ButterKnife进行实现的。
>
这里我用AndroidStudio根据ButterKnife的版本进行了实现。

文字枯燥，还是先看下butterknife的module图：
![这里写图片描述](http://img.blog.csdn.net/20160401092206024)

先看下ButterKnife中生成的java代码（看看骚包的注释 Do not modify!）

{% highlight java %}
// Generated code from Butter Knife. Do not modify!
// Generated code from Butter Knife. Do not modify!
package com.example.butterknife;

import android.view.View;
import android.widget.AdapterView;
import butterknife.ButterKnife;
import butterknife.internal.DebouncingOnClickListener;
import butterknife.internal.Finder;
import butterknife.internal.Utils;
import butterknife.internal.ViewBinder;
import java.lang.IllegalStateException;
import java.lang.Object;
import java.lang.Override;
import java.lang.SuppressWarnings;

public class SimpleActivity$$ViewBinder<T extends SimpleActivity> implements ViewBinder<T> {
  @Override
  public void bind(final Finder finder, final T target, Object source) {
    Unbinder unbinder = createUnbinder(target);
    View view;
    view = finder.findRequiredView(source, 2130968576, "field 'title'");
    target.title = finder.castView(view, 2130968576, "field 'title'");
    view = finder.findRequiredView(source, 2130968577, "field 'subtitle'");
    target.subtitle = finder.castView(view, 2130968577, "field 'subtitle'");
    view = finder.findRequiredView(source, 2130968578, "field 'hello', method 'sayHello', and method 'sayGetOffMe'");
    target.hello = finder.castView(view, 2130968578, "field 'hello'");
    unbinder.view2130968578 = view;
    view.setOnClickListener(new DebouncingOnClickListener() {
      @Override
      public void doClick(View p0) {
        target.sayHello();
      }
    });
    view.setOnLongClickListener(new View.OnLongClickListener() {
      @Override
      public boolean onLongClick(View p0) {
        return target.sayGetOffMe();
      }
    });
    view = finder.findRequiredView(source, 2130968579, "field 'listOfThings' and method 'onItemClick'");
    target.listOfThings = finder.castView(view, 2130968579, "field 'listOfThings'");
    unbinder.view2130968579 = view;
    ((AdapterView<?>) view).setOnItemClickListener(new AdapterView.OnItemClickListener() {
      @Override
      public void onItemClick(AdapterView<?> p0, View p1, int p2, long p3) {
        target.onItemClick(p2);
      }
    });
    view = finder.findRequiredView(source, 2130968580, "field 'footer'");
    target.footer = finder.castView(view, 2130968580, "field 'footer'");
    target.headerViews = Utils.listOf(
        finder.<View>findRequiredView(source, 2130968576, "field 'headerViews'"), 
        finder.<View>findRequiredView(source, 2130968577, "field 'headerViews'"), 
        finder.<View>findRequiredView(source, 2130968578, "field 'headerViews'"));
    target.unbinder = unbinder;
  }

  @SuppressWarnings("unchecked")
  protected <U extends Unbinder<T>> U createUnbinder(T target) {
    return (U) new Unbinder(target);
  }

  @SuppressWarnings("unchecked")
  protected <U extends Unbinder<T>> U accessUnbinder(T target) {
    return (U) target.unbinder;
  }

  public static class Unbinder<T extends SimpleActivity> implements ButterKnife.ViewUnbinder<T> {
    private T target;

    View view2130968578;

    View view2130968579;

    protected Unbinder(T target) {
      this.target = target;
    }

    @Override
    public final void unbind() {
      if (target == null) throw new IllegalStateException("Bindings already cleared.");
      unbind(target);
      target = null;
    }

    protected void unbind(T target) {
      target.title = null;
      target.subtitle = null;
      view2130968578.setOnClickListener(null);
      view2130968578.setOnLongClickListener(null);
      target.hello = null;
      ((AdapterView<?>) view2130968579).setOnItemClickListener(null);
      target.listOfThings = null;
      target.footer = null;
      target.headerViews = null;
      target.unbinder = null;
    }
  }
}
{% endhighlight %}

## [ButterKnifeView注入demo(简单易学)](http://download.csdn.net/detail/yuanyang5917/9478759)

>这个demo的代码生成，以及接口思想，以及调用生成代码的思想，都是借鉴ButterKnife的

>我先简单实现了ButterKnife的View注入，去除了ButterKnife繁琐的编译校验，本篇也没有添加事件注入(下一篇会有)

### XUtils 与 ButterKnife实现方式的区别
上篇XUtils实现注解初始化用的是反射 +  动态代理

>JAVA反射机制是在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意一个方法和属性；这种动态获取的信息以及动态调用对象的方法的功能称为java语言的反射机制。

## APT
而ButterKnife使用的是APT (Annotation Processing Tool )
>apt是一个命令行工具，与之配套的还有一套用来描述程序语义结构的Mirror API。Mirror API（com.sun.mirror.*）描述的是程序在编译时刻的静态结构。通过Mirror API可以获取到被注解的Java类型元素的信息，从而提供相应的处理逻辑。具体的处理工作交给apt工具来完成。编写注解处理器的核心是AnnotationProcessorFactory和AnnotationProcessor两个接口。后者表示的是注解处理器，而前者则是为某些注解类型创建注解处理器的工厂。

1. APT配置
需要再compiler Module中的main.resources.META-INF.services下新建文本java.annotation.processing.Processor，在文本内添加Processor全名(我的Processor是com.example.MyProcessor)
或者直接在Processor实现类上@AutoService(Processor.class)(我还没实现成功)
需要在build.gradle里面配置APT插件(classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8')

2. Processor类
(1)JAVA1.6以后的Processor实现类都需要extends AbstractProcessor
(2)Processor类需要指定其处理的注解
方式一(类上注解)：@SupportedAnnotationTypes("com.example.AptBetter")
方式二(重写方法)：getSupportedAnnotationTypes
(4)process方法，编译时进入的主方法，用于分析注解信息，以及生成代码
(4)Processor都要指定版本
一般获取最新的版本：

{% highlight java %}
@Override
public SourceVersion getSupportedSourceVersion() {
return SourceVersion.latestSupported();
}
{% endhighlight %}

3. Process内部元素
ProcessingEnvironment：编译环境
{% highlight java %}
// 用于生成代码
Filer filer = processingEnv.getFiler();
// 用于编译时给图
Messager messager = processingEnv.getMessager();
// Element工具，可以获取包名
Elements elementUtils = processingEnv.getElementUtils();
{% endhighlight %}

RoundEnvironment：回合环境(国内暂时没有固定的说法，暂且定义为元素环境)
因为可以从中获取到5大Element
{% highlight ios %}
Mirror的五大Element
PackageElement 包元素
TypeElement     类元素
VariableElement  变量元素
ExecutableElement  方法元素
TypeParameterElement  参数元素
通过这些元素，可以获取到元素的各种信息：注解、类型、名称等等
{% endhighlight %}
其中TypeMirror.accept这个我还很模糊，有明白的请赐教

4. 代码生成器
这里有一个大杀器：JavaPoet (Java诗人) 也是Jake Wharton所参与的著作之一，膜拜Jake Wharthon，膜拜Square。大家可以直接到Github上搜索JavaPoet。


## 运行期流程

![这里写图片描述](http://img.blog.csdn.net/20160401155957286)

主要是通过Butterknife调用生成的Activity$$ViewBinder类中的bind方法，实现了控件初始化、时间注入

### Question :
(1)在代码编写的时候类都还没有生成，那么如何获取其对象
answer:通过反，Activity也是通过这种方式生成的
{% highlight java %}
Class<?> viewBindingClass = Class.forName(clsName + "$$ViewBinder");
ViewBinder viewBinder = (ViewBinder<Object>) viewBindingClass.newInstance();
{% endhighlight %}
(2)如何能调用bind方法，如何能强转，因为生成的代码实现了ViewBinder接口，接口的力量啊！针对接口编程！

## 分析compiler

思路分析：
（1）一个类、一个内部类生成一个同包下的名称相似的IOC容器类
（2）一类中会有很多控件需要初始化，那么我们把一个类、一个内部类所提供的多个控件信息、类名、包名、id封装成一个Bean：BindingClass(ButterKnife除了可以给控件初始化之外，还可以给各种资源初始化，甚至可以解绑UnBinding，ButterKnife中把每种类型都封装成一种Bean，这里我们紧紧做了View的初始化),我们这里BindingClass里面仅仅放的是FieldViewBinding
（3）一个需要初始化的类对应一个BindingClass(含有很多FieldViewBinding)，对应生成一个ViewBinder，那么生成代码的逻辑我们可以放在Compiler中的BindingClass，作为其内部类(可以直接获取BindingClass中的类名、包名、控件集合)


![这里写图片描述](http://img.blog.csdn.net/20160404004902680)

## 编译器读信息


{% highlight java %}
// 所有AptBetter的成员变量
        Set<? extends Element> elementsAnnotatedWith = roundEnv.getElementsAnnotatedWith(AptBetter.class);
        // 元素归类
        for (Element element : elementsAnnotatedWith) {
            TypeElement typeElement = (TypeElement) element.getEnclosingElement();
            BindingClass bindingClass = null;
            if (allBindEvent.containsKey(typeElement)) {
                bindingClass = allBindEvent.get(typeElement);
            } else {
                String targetType = typeElement.getQualifiedName().toString();
                String classPackage = getPackageName(typeElement);
                // packageName$className$$ViewBinder
                String className = getClassName(typeElement, classPackage) + SUFFIX;

                bindingClass = new BindingClass(classPackage, className, targetType);
                allBindEvent.put(typeElement, bindingClass);
            }
            int value = element.getAnnotation(AptBetter.class).value();
            String elementName = element.getSimpleName().toString();
            TypeName typeName = TypeName.get(element.asType());
            bindingClass.addField(value, new FieldViewBinding(elementName, typeName, value));
        }
{% endhighlight %}

## 代码生成(JavaPoet)
这里是用的是JavaPoet，下面是教程地址：

- [JavaPoet In Github](https://github.com/square/javapoet)
- [JavaPoet博客教程](http://p.codekk.com/detail/Android/square/javapoet)