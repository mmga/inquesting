# AnnotationProcessorDemo

[TOC]

## Annotation
### 元注解
元注解的作用就是负责注解其他注解。
 1. @Target
 2. @Retention 
 3. @Documented
 4. @Inherited

#### @Target：
说明了Annotation所修饰的对象的作用：用户描述注解的使用范围
取值(ElementType):

> ElementType.CONSTRUCTOR:构造器声明
ElementType.FIELD:成员变量、对象、属性（包括enum实例）
ElementType.LOCAL_VARIABLE:局部变量声明
ElementType.METHOD:方法声明
ElementType.PACKAGE:包声明
ElementType.PARAMETER:参数声明
ElementType.TYPE:类、接口（包括注解类型)或enum声明

#### @Retention 
表示需要在什么级别保存该注释信息，用于描述注解的生命周期
取值(RetentionPolicy):
> RetentionPolicy.SOURCE:在源文件中有效
RetentionPolicy.CLASS:在编译时有效
RetentionPolicy.RUNTIME:在运行时有效

#### @Documented
标记注解，没有成员
用于描述其它类型的annotation应该被作为标注的程序成员的公共api，可以文档化

#### @Inherited
标记注解
用该注解修饰的注解，会被子类继承

**举个栗子**�
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.CLASS)
public @interface Mmga {
}
```
表示只能注解在class、interface、和enum上

----

## APT

> APT(Annotation Processing Tool)是一种处理注释的工具,它对源代码文件进行检测找出其中的Annotation，使用Annotation进行额外的处理。

### 工程概览
![](http://osx5yzuma.bkt.clouddn.com/image/apt1.png)

### 创建 Annotation Module
首先，我们需要新建一个名称为annotation的 **Java Library**。
![](http://osx5yzuma.bkt.clouddn.com/apt2.png)

#### 配置 build.gradle 
主要是规定 jdk 版本
```groovy
apply plugin: 'java'

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
}

sourceCompatibility = "1.7"
targetCompatibility = "1.7"
```

### 创建 Compiler Module
创建一个名为compiler的**Java Library**，这个类将会写代码生成的相关代码。核心就是在这里。

#### 配置 build.gradle
```groovy
apply plugin: 'java'
sourceCompatibility = 1.7 
targetCompatibility = 1.7 
dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile 'com.google.auto.service:auto-service:1.0-rc2'
    compile 'com.squareup:javapoet:1.7.0'
    compile project(':annotation')
}
```
 1. 定义编译的jdk版本为1.7。
 2. AutoService 主要的作用是注解 processor 类，并对其生成 META-INF 的配置信息。
 3. JavaPoet 这个库的主要作用就是帮助我们通过类调用的形式来生成代码。
 4. 依赖上面创建的annotation Module。



#### 创建 Processor 类，编写生成代码相关的逻辑
```java
@AutoService(Processor.class)
public class HelloWorldProcessor extends AbstractProcessor {

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        //生成代码相关逻辑
        return false;
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        return Collections.singleton(HelloWorld.class.getCanonicalName());
    }
}
```


**举个栗子**
要生成的类 HelloWorld
```java
package com.example.helloworld;

public final class HelloWorld {
  public static void main(String[] args) {
    System.out.println("Hello, JavaPoet!");
  }
}
```
修改Processor类中的 process 方法
```java
@Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        MethodSpec main = MethodSpec.methodBuilder("main")
                .addModifiers(Modifier.PUBLIC, Modifier.STATIC)
                .returns(void.class)
                .addParameter(String[].class,"args")
                .addStatement("$T.out.print($S)", System.class, "Hello,JavaPoet!")
                .build();
        TypeSpec helloWorld = TypeSpec.classBuilder("HelloWorld")
                .addModifiers(Modifier.PUBLIC, Modifier.FINAL)
                .addMethod(main)
                .build();
        JavaFile javaFile = JavaFile.builder("com.example.helloworld", helloWorld)
                .build();

        try {
            javaFile.writeTo(processingEnv.getFiler());
        } catch (IOException e) {
            e.printStackTrace();
        }


        return false;
    }
```

### 在app中使用
#### 配置 app 的 build.gradle
```groovy
dependencies {
// ..
compile project(':annotation')
    annotationProcessor project(':compiler')
    }
```

#### 添加注解
```java
@HelloWorld
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }
}
```

点击 make project 可以在 app 的 build build/generated/source/apt 看到生成的代码。

更多栗子
[通过 annotation processor 生成弹出 Toast](https://github.com/mmga/AnnotationProcessorDemo/blob/master/compiler/src/main/java/com/example/ToastProcessor.java)

参考资料 
[javapoet](http://www.jianshu.com/p/95f12f72f69a)  

[Android APT（编译时代码生成）最佳实践](https://github.com/taoweiji/DemoAPT?utm_source=tuicool&utm_medium=referral)  

[APT实用案例一：状态模式之就算违背开闭原则又何妨？](http://blog.csdn.net/drd_zsd123/article/details/72822662)  

[Android公共技术点之一-Java注解](http://yeungeek.com/2016/04/25/Android%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%E4%B8%80-Java%E6%B3%A8%E8%A7%A3/)  









