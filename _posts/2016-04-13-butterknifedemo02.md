---
layout: post
title: "注解初始化控件(ButterKnife方式)<下>"
date: 2016-04-13 00:00:00 +0800
cover: false
tags: android
subclass: 'post tag-fiction'
categories: 'casper'
cover: 'assets/images/cover6.jpg'
navigation: True
logo: 'assets/images/ghost.png'

---


一个星期没更新博客了，虽然目前博客很乱！最近比较忙，视力有些下降，不过ButterKnife的事件实现终算完成了！

>### [ButterKnifeDemo实现(注解完善，方便阅读)](http://download.csdn.net/detail/yuanyang5917/9489865)

## 目标
之所以butterknife可以实现点击view的时候调用注解过的方法，其实是在点击的回调方法中调用目标类的相应注释过的方法：
{% highlight java %}
        view.setOnClickListener(new DebouncingOnClickListener() {
            @Override
            public void doClick(View v) {
                target.click(v);
            }
        });
{% endhighlight %}

拿我的demo为例，我们最终是要生成一个如下的类：


这里写代码片

{% highlight java %}
package com.turbo.demo.apt;

import android.view.View;

import com.turbo.apt.library.internal.DebouncingOnClickListener;
import com.turbo.apt.library.internal.Finder;
import com.turbo.apt.library.internal.ViewBinder;

/**
 * Created by LuckyTurbo on 16/4/11.
 *
 * 待生成的类
 */
public class MainActivity_ViewBinder<T extends MainActivity> implements ViewBinder<T> {
    @Override
    public void bind(Finder finder, final T target, Object source) {
        View view = null;
        view = finder.findRequiredView(source,R.id.textView,"textView 我 啊");
        target.textView = finder.castView(view,R.id.textView,"textView 我 啊");
        view = finder.findRequiredView(source,R.id.rightView,"rightView 我 啊");
        target.rightView = finder.castView(view,R.id.rightView,"rightView 我 啊");
        view.setOnClickListener(new DebouncingOnClickListener() {
            @Override
            public void doClick(View v) {
                target.click(v);
            }
        });

    }
}
{% endhighlight %}

那么下面要做的就是收集各种注解数据，然后再根据这些数据生成最终的类。

## 收集数据
绝大部分方法的数据都是从注解中获取的，那么我们如何设计注解呢？
### 注解设计分析
需求：
1.id
2.设置方法 (example:setOnLongClickListener)
3.接口类型 (android.view.View.OnLongClickListener)
4.接口中的方法：
1.方法名 (onLongClick)
2.参数类型 (android.view.View)
3.方法返回值 （boolean）

这么多东西如果在每个注解里面都加上，想想都恶心，但是注解并没有继承这一说，但是我们可以在注解的基础上再进行注解，上层注解里面可以设定默认值，每次给上层注解填值就可以了
——其实我们想要的就是规范，就是模版，其实就是针对规范编程

下面看下主要的几个注解：

- @OnClick

{% highlight java %}
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.CLASS)
@ListenerClass(
        targetType = "android.view.View",
        setter = "setOnClickListener",
        type = "com.turbo.apt.library.internal.DebouncingOnClickListener",// 这个是butterKnife中的防卡点击
        method = @ListenerMethod(
                name = "doClick",
                parameters = "android.view.View",
                returnType = "void"
        )
)
public @interface OnClick {
    // ids annotation注解 奇怪没导入成功
    int[] value() default {View.NO_ID};
}
{% endhighlight %}
- @ListenerClass

{% highlight java %}
@Target(ElementType.ANNOTATION_TYPE)
@Retention(RetentionPolicy.RUNTIME)// 靠，获取注解的注解好像必需要使用runtime，不然取不到
public @interface ListenerClass {
    //    @ListenerClass(
    //            targetType = "android.view.View",
    //            setter = "setOnClickListener",
    //            type = "butterknife.internal.DebouncingOnClickListener",
    //            method = @ListenerMethod(
    //                    name = "doClick",
    //                    parameters = "android.view.View"
    //            )
    //    )

    // 某view
    String targetType();
    // 设置方法的名称
    String setter();
    // 接口全称
    String type();
    /** Enum which declares the listener callback methods. Mutually exclusive to {@link #method()}. */
    // 跟method互斥
    Class<? extends Enum<?>> callbacks() default NONE.class;
    /**
     * Method data for single-method listener callbacks. Mutually exclusive with {@link #callbacks()}
     * and an error to specify more than one value.
     */
    ListenerMethod method();

    // callback的默认值
    enum NONE {}
}
{% endhighlight %}
- @ListenerMethod

{% highlight java %}
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ListenerMethod {

    // listener 方法 的 名字
    String name();

    // 方法参数
    String[] parameters() default {};

    // 方法返回类型
    String returnType() default "void";

    /** If {@link #returnType()} is not {@code void} this value is returned when no binding exists. */
    // 如果returnType 不是void，就返回这个值
    String defaultReturn() default "null";
}
{% endhighlight %}

### 注解收集数据结构
![这里写图片描述](http://img.blog.csdn.net/20160413140458623)

### 注解收集数据逻辑

{% highlight java %}
private Map<TypeElement, VBinderBuilder> findAndParseTargets(RoundEnvironment roundEnv) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        Map<TypeElement, VBinderBuilder> vBinderBuilderMap = new LinkedHashMap<>();

        // 解析所有的@Bind 元素
        for (Element element : roundEnv.getElementsAnnotatedWith(Bind.class)) {
            TypeElement typeElement = (TypeElement) element.getEnclosingElement();
            VBinderBuilder builderClass = null;
            if (vBinderBuilderMap.containsKey(typeElement)) {
                builderClass = vBinderBuilderMap.get(typeElement);
            } else {
                String targetType = typeElement.getQualifiedName().toString();
                String classPackage = getPackageName(typeElement);
                // packageName$className_ViewBinder
                String className = getClassName(typeElement, classPackage) + APTConfig.SUFFIX;

                builderClass = new VBinderBuilder(classPackage, className, targetType);
                vBinderBuilderMap.put(typeElement, builderClass);
            }

            int value = element.getAnnotation(Bind.class).value();
            String elementName = element.getSimpleName().toString();
            TypeName typeName = TypeName.get(element.asType());
            FieldViewBinding fieldViewBinding = new FieldViewBinding(elementName, typeName, value);
            builderClass.addField(value, fieldViewBinding);
        }
        // 解析所有LISTENERS：方法上的事件注解
        for (Class<? extends Annotation> annotationClass : LISTENERS) {
            for (Element element : roundEnv.getElementsAnnotatedWith(annotationClass)) {
                // 方法元素
                ExecutableElement executableElement = (ExecutableElement) element;
                // 类元素(持有者元素)
                TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();
                // 方法的注解对象
                Annotation annotation = element.getAnnotation(annotationClass);
                // 注解的value方法
                Method annotationValue = annotationClass.getDeclaredMethod("value");
                // 获取注解的ids
                int[] ids = (int[]) annotationValue.invoke(annotation);
                // 方法名称
                String name = executableElement.getSimpleName().toString();
                // 注解的注解
                ListenerClass listenerClass = annotationClass.getAnnotation(ListenerClass.class);
                // 注解注解的参数
                ListenerMethod listenerMethod = listenerClass.method();
                // 方法参数
                List<? extends VariableElement> methodParameters = executableElement.getParameters();

                Parameter[] parameters = Parameter.NONE;
                // 真实方法参数
                if (!methodParameters.isEmpty()) {
                    parameters = new Parameter[methodParameters.size()];
                    // 注解的注解中的参数类型
                    String[] parameterTypes = listenerMethod.parameters();

                    for (int i = 0; i < methodParameters.size(); i++) {
                        VariableElement methodParameter = methodParameters.get(i);
                        // 方法参数
                        TypeMirror methodParameterType = methodParameter.asType();
                        if (methodParameterType instanceof TypeVariable) {
                            TypeVariable typeVariable = (TypeVariable) methodParameterType;
                            methodParameterType = typeVariable.getUpperBound();
                            messager.printMessage(Diagnostic.Kind.ERROR, ((TypeVariable) methodParameterType).asElement().getSimpleName());
                        }
                        for (int j = 0; j < parameterTypes.length; j++) {
                            // 封装了真实参数的位置，类型
                            parameters[j] = new Parameter(j, TypeName.get(methodParameterType));
                        }
                    }
                }
                // name:方法名 parameters:方法参数parameter required:方法上是否有Optional注解（一般不会有，所以为true）
                // 一般都是true，我们这里没有给注解设置Optional注解
                MethodViewBinding methodViewBinding = new MethodViewBinding(name, Arrays.asList(parameters), true);
                VBinderBuilder builderClass = getOrCreateTargetClass(vBinderBuilderMap, enclosingElement);
                for (int id : ids){
                    builderClass.addMethod(id,listenerClass,listenerMethod,methodViewBinding);
                }
            }
        }
        return vBinderBuilderMap;
    }
{% endhighlight %}

## ViewBinder绑定帮助类生成

根据上一篇的实现步骤，我们生成一个实现ViewBinder接口，及其方法，实现控件初始化的类已经不是一件难事，现在最大的困难就是如何通过JavaPoet实现类似下面的代码：
{% highlight java %}
        view.setOnClickListener(new DebouncingOnClickListener() {
            @Override
            public void doClick(View v) {
                target.click(v);
            }
        });
{% endhighlight %}

根据butterKnife代码我给出具体的解决步骤：
以@OnLongClick为例：
{% highlight java %}
// new DebouncingOnClickListener()
// 这里就是一个空接口以及一个父类
TypeSpec.Builder interfaceEventBuilder = TypeSpec.anonymousClassBuilder("")
						.superclass(ClassName.bestGuess(keyListenerClass.type()));
{% endhighlight %}

{% highlight java %}
// 这里代表的是 方法：public void doClick
MethodSpec.Builder interfaceMethodBuilder = MethodSpec.methodBuilder(method.name())
								.addAnnotation(Override.class)
								.addModifiers(Modifier.PUBLIC)
								.returns(bestGuess(method.returnType()));
{% endhighlight %}

{% highlight java %}
// 第一个参数  是参数类型  第一个参数是参数名(p0 p1 p2 ...)
// 方法中的参数(View v)
interfaceMethodBuilder.addParameter(bestGuess(parameterTypes[i]), "p" + i);
{% endhighlight %}

{% highlight java %}
// 调用target的对应方法：target.click(v);
codeBuilder.add("target.$L(", methodViewBinding.getName());
{% endhighlight %}

整个方法的生成代码如下：

{% highlight java %}
    private void addMethodBindings(MethodSpec.Builder resultBuilder, VBinderData vbinderData) {
        Map<ListenerClass, Map<ListenerMethod, Set<MethodViewBinding>>> listenerMethodBindings = vbinderData.getMethodBindings();
        if (listenerMethodBindings.isEmpty()) {
            return;
        }
        Set<Map.Entry<ListenerClass, Map<ListenerMethod, Set<MethodViewBinding>>>> entries = listenerMethodBindings.entrySet();
        for (Map.Entry<ListenerClass, Map<ListenerMethod, Set<MethodViewBinding>>> entry : entries) {
            ListenerClass keyListenerClass = entry.getKey();
            Map<ListenerMethod, Set<MethodViewBinding>> methodValue = entry.getValue();
            // 创建接口实体类(空匿名类的接口父类)
            // anonymous 匿名
            TypeSpec.Builder interfaceEventBuilder = TypeSpec.anonymousClassBuilder("")
                    .superclass(ClassName.bestGuess(keyListenerClass.type()));
            // 根据ListenerClass 中的 参数 生成代码
            ListenerMethod method = keyListenerClass.method();
            if (method != null) {
                MethodSpec.Builder interfaceMethodBuilder = MethodSpec.methodBuilder(method.name())
                        .addAnnotation(Override.class)
                        .addModifiers(Modifier.PUBLIC)
                        .returns(bestGuess(method.returnType()));
                String[] parameterTypes = method.parameters();
                for (int i = 0; i < parameterTypes.length; i++) {
                    // 第一个参数  是参数类型  第一个参数是参数名(p0 p1 p2 ...)
                    interfaceMethodBuilder.addParameter(bestGuess(parameterTypes[i]), "p" + i);
                }

                boolean hasReturnType = !"void".equals(method.returnType());
                // 代码块
                CodeBlock.Builder codeBuilder = CodeBlock.builder();
                if (hasReturnType) {
                    codeBuilder.add("return ");
                }

                // 一个id对应多个ListenerMethod
                if (methodValue.containsKey(method)) {
                    for (MethodViewBinding methodViewBinding : methodValue.get(method)) {
                        // 调用target的对应方法
                        codeBuilder.add("target.$L(", methodViewBinding.getName());
                        List<Parameter> parameters = methodViewBinding.getParameters();
                        // MethodListener注解的参数
                        String[] methodParameters = method.parameters();
                        // 优化了每次parameters.size()
                        for (int i = 0, count = parameters.size(); i < count; i++) {
                            if (i > 0) {
                                codeBuilder.add(", ");
                            }
                            Parameter parameter = parameters.get(i);
                            int listenerPosition = parameter.getListenerPosition();
                            // 类型不一样，范型万能转换
                            if (parameter.requiresCast(methodParameters[i])) {
                                codeBuilder.add("finder.<$T>castParam(p$L, $S, $L, $S, $L)\n", parameter.getType(),
                                        listenerPosition, method.name(), listenerPosition, methodViewBinding.getName(), i);
                            } else {
                                codeBuilder.add("p$L", listenerPosition);
                            }
                        }
                        codeBuilder.add(");\n");
                    }
                } else if (hasReturnType) {
                    codeBuilder.add("$L;\n", method.defaultReturn());
                }
                interfaceMethodBuilder.addCode(codeBuilder.build());
                interfaceEventBuilder.addMethod(interfaceMethodBuilder.build());
            }
            // 如果不是view类型，需要强转
            if (!APTConfig.VIEW_TYPE.equals(keyListenerClass.targetType())) {
                // targetType不是view的时候需要强转
                resultBuilder.addStatement("(($T) view).$L($L)", bestGuess(keyListenerClass.targetType()),
                        keyListenerClass.setter(), interfaceEventBuilder.build());
            } else {
                // view.setOnClickListener(对象)
                resultBuilder.addStatement("view.$L($L)", keyListenerClass.setter(), interfaceEventBuilder.build());
            }
        }
    }
{% endhighlight %}
其他的具体步骤大家看下demo

## 总结
代码这个东西，看跟敲是两种体验，也是两种结果，以后会继续坚持，总结下从butterKnife框架细节中我学到的知识：

- BitSet : java中提供的此神器用来记录我对一组数据的改动与否
- @Retention(RUNTIME) ：注解的注解是需要runtime的，即便仅在编译期读取，RetentionPolicy.CLASS也是不奏效的，必需是RetentionPolicy.RUNTIME
-  for (int i = 0, count = parameters.size(); i < count; i++) : 省去每次遍历调用parameters.size()的同时，又优雅的初始化了count并附上了想要的值(智慧而优雅)