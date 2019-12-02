title: Java注解学习(二)
date: 2016-07-03 21:08:20
tags:
---

上一篇博客关注点在利用反射解析运行时的注解，这篇学习下利用apt来解析编译时的注解。

---
## APT
**概念**
`Annotation Processor Tool`(注解处理器工具)是`javac`的一个工具，它用来在编译时去扫描和处理注解。你可以自定义一些注解，然后注册相应的注解处理器来处理你的自定义注解。

**用途**
利用apt来解析编译时注解的作用通常是用来生成文件，比如可以用来生成`.java`代码，而且apt是在javac编译代码之前，意味着我们生成的代码也会被编译。

<!-- more -->

---
## 编写注解处理器

### 自定义注解

首先我们先定义一个存活时间为编译时的注解。
```
package annotation;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.CLASS)
public @interface CompileAnnotation {
    String value() default "";
}
```

### 编写Processor

然后来编写`注解处理器`来解析我们的自定义注解。

首先，我们需要知道注解处理器是在它自己的JVM里面运行，javac会启动一个完整的JVM来运行注解处理器。意味着我们像写其他Java程序一样，使用你想用的类库。

写法方面，所有的注解处理器都需要去继承`AbstractProcessor`。
```
public class CompileAnnotationProcessor extends AbstractProcessor {

    /**
     * 初始化注解处理器环境,ProcessingEnvironment提供了很多有用的工具类.比如Elements,Types,Filer
     */
    @Override
    public synchronized void init(ProcessingEnvironment env){ }

    /**
     * 处理注解的主要方法, 我们需要重写这个方法来扫描,处理注解,生成文件.
     * RoundEnvironment包含了被注解的元素,包括类,方法,实例域等.
     * annotations包含了所有的注解信息.
     */
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment env) { }

    /**
     * 指定这个注解处理器是去处理哪些注解的.
     * 换句话说,指定注解处理器注册到哪些注解上
     * 可以通过@SupportedAnnotationTypes()注解来替代
     */
    @Override
    public Set<String> getSupportedAnnotationTypes() { }

    /**
     * 指定使用的Java版本,通常返回SourceVersion.latestSupported()
     * 可以通过@SupportedSourceVersion(SourceVersion.latestSupported())注解来替代
     */
    @Override
    public SourceVersion getSupportedSourceVersion() { }

}
```

这里我们来重写这个`process()`方法。
如一开始提到的，`编译时注解`主要是用来生成文件，比如java代码的。这里为方便理解，就生成一个文本文件就好。最后生成的文件的文件名为`被注解的类的类名`,文件内容为`注解的值。

```
@SupportedAnnotationTypes({"annotation.CompileAnnotation"})
@SupportedSourceVersion(SourceVersion.RELEASE_8)
public class CompileAnnotationProcessor extends AbstractProcessor {
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        //遍历所有被注解了@CompileAnnotation的元素,因为这个注解类型是Type,所以这里只能是类这种元素.
        for (Element e : roundEnv.getElementsAnnotatedWith(CompileAnnotation.class)) {
            CompileAnnotation annotation = e.getAnnotation(CompileAnnotation.class);
            try {
                //以被注解的类名作为文件名,我将其放到E盘了.
                File file = new File("E:\\" + e.getSimpleName() + ".txt");
                FileWriter fileWriter = new FileWriter(file);
                fileWriter.append(annotation.value());
                fileWriter.flush();
                fileWriter.close();
            } catch (IOException e1) {
                e1.printStackTrace();
            }
        }

        return true;
    }
}
```

### 注册处理器

写好了注解处理器之后，需要将注解处理器注册到javac中，为此，我们需要提供一个jar包，意味着我们要把这个项目打成jar包。

在此之前，还要配置一个文件,需要再`resoures`文件夹下面创建个`META-INF`文件夹，再里面创建`services`文件夹，再里面创建`javax.annotation.processing.Processor`文件，文件的内容是注解处理器类的全名,每个类占一行。
```
processor.CompileAnnotationProcessor
```
如果不想创建这个文件,可以用Google的[AutoService](https://github.com/google/auto/tree/master/service)，在注解处理器上加上`@AutoService(Processor.class)`,便可以自动生成这个文件。


最后，项目结构应该是这样的:
![图](/uploads/processor-project.jpg)

然后把这个项目打成jar包，这样我们的处理注解器就算写完了。

#### 使用编译时注解

然后我们再新建一个项目,将刚才的jar包导入进来。

我这个编写2个Java类文件来做测试。
```
package Package1;

import annotation.CompileAnnotation;

@CompileAnnotation(value = "this is test value")
public class Test {
}


package Package2;

import annotation.CompileAnnotation;

@CompileAnnotation(value = "this is test1 value")
public class Test1 {
}
```

然后编译这个新的项目，就会发现E盘生成了2个文本文件,Test.txt和Test1.txt。内容为注解的值。

编写一个编译时注解的处理器大致过程就是这样。但这里只示范了创建文本文件。
但绝大部分情况是需要生成Java代码的，通常使用[javapoet](https://github.com/square/javapoet)来生成Java源代码。

更详细的内容请看这篇[文章](http://hannesdorfmann.com/annotation-processing/annotationprocessing101),也有哥们翻译成中文了, [中文版](http://www.race604.com/annotation-processing/)

---

## 参考文档
[注解处理器](http://hannesdorfmann.com/annotation-processing/annotationprocessing101)