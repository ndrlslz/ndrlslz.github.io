title: Java注解学习(一)
date: 2016-06-30 21:03:03
tags:
---


## 注解基础知识

### 概念

之前在学习Python的时候，有用到python里面的注解(应该叫装饰器)， 它实际上是对函数的包装。
Java里面的注解概念和Python还是不一样。使用Java的注解相当于是在代码中嵌入一些元数据，然后利用编译器，反射等机制来解析它。

### 分类

**元注解**
元注解的作用是来负责对注解进行标注，用来定义注解的。

<!-- more -->

| 元注解     | 作用 |
| ----       | ---    |
| Documented | 被注解的元素会被javadoc或其他工具文档化 |
| Inherited  | 表示该注解会被自动继承，默认为False |
| Target     | 表示该注解可以修饰哪些程序元素，比如修饰类，方法，实例域等，默认可以修饰所有 |
| Retention  | 表示该注解的被保留的时间，默认为CLASS(编译时) |


**标准注解**


| 内置注解    | 作用 |
| ---         | ---  |
| Override    | 表示重写父类的方法 |
| Deprecated  | 表示该元素不推荐使用 |
| SuppressWarnings | 表示忽略警告 |


**自定义注解**
按照需要可以定义自己的注解，通常在框架层面大量的用到自定义注解。
下面是自定义注解的例子:
```
@Documented
@Inherited
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface MyAnnotation {
    String value() default "";
}
```

---
## 解析运行时注解
定义了注解之后当然需要解析它。而解析的方式也根据注解的Retention类型来用不同的方式。

| Retention类型  | 作用 |
| ---            | ---  |
| SOURCE         | 表示只保留到源代码时期，经过编译后，便丢弃。通常用来告诉编译器一些信息，例如Java自带的@SuppressWarnings，就是告诉编译器压制警告，我们自定义注解的时候一般不会用到这个类型 |
| CLASS          | 表示保留到字节码时期，编译后还会存在，但是运行的时候就不存在了。解析这种注解需要用到apt(Annotation Processing Tool)。 |
| RUNTIME        | 表示会保留到运行时，这种注解通过反射来解析它。 |

这篇博客先学习下通过反射来解析运行时的注解。
在通过反射获得了class，method，field等对象后，通过反射可以获得注解。 例子如下：
```
Class clazz = ... //反射获得Class对象
clazz.getAnnotation(AnnotationName.class);      //获得这个类上的某个指定的注解
clazz.getAnnotations();                         //获得这个类的所有注解
clazz.isAnnotationPresent(AnnotationName.class);//查看这个类是否有这个注解
```
其他method和field级别的注解获取方式一样。

我们利用上面定义的`MyAnnotation`的注解来实际用一下。
下面是个例子，`Application`这个类使用了`MyAnnotation`的注解，然后我们在main方法里通过反射来获得注解的value。
```
package Annotation;


@MyAnnotation(value = "this is annotation value")
class Application {

}

public class Test {
    public static void main(String[] args) throws ClassNotFoundException {
        Class clazz = Class.forName("Annotation.Application");
        MyAnnotation annotation = (MyAnnotation) clazz.getAnnotation(MyAnnotation.class);
        if (annotation != null) {
            System.out.println(annotation.value());
        }
    }
}

//this is annotation value
```

---
## 模拟SpringMVC的handleMapping
在用SpringMVC写web应用的时候，通常定义一个`Controller`，并且加上`@RequestMapping`注解来为这个controller添加它可以处理的url。当用户请求的url到达服务器，通过handleMapping来找到对应的controller里面的方法去处理。

我这里通过运行时注解来简单模拟一下这个过程。篇幅有限，代码尽量最小化了。

先定义`Controller`和`RequestMapping`注解。
```
package Annotation;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Controller {
}

@Retention(RetentionPolicy.RUNTIME)
public @interface RequestMapping {
    String value() default "";
}
```

然后创建个controller的业务类来使用这些注解。
```
package controller;

import Annotation.Controller;
import Annotation.RequestMapping;

@Controller
@RequestMapping(value = "/main")
public class MainController {

    @RequestMapping(value = "/login")
    public void login() {
        System.out.println("login successfully");
    }

    @RequestMapping(value = "/logout")
    public void logout() {
        System.out.println("logout successfully");
    }
}
```

最后利用反射来解析controller类，将url和controller加上映射关系。
```
public class Test {

    /**
     * 模拟Spring的component-scan的配置
     * 给定一个package，扫描package下面的所有类，并获得所有Class对象
     */
    private static List<Class> getClasses(String packageName)
            throws ClassNotFoundException, IOException {
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
        assert classLoader != null;
        String path = packageName.replace('.', '/');
        Enumeration<URL> resources = classLoader.getResources(path);
        List<File> dirs = new ArrayList<>();
        while (resources.hasMoreElements()) {
            URL resource = resources.nextElement();
            dirs.add(new File(resource.getFile()));
        }
        ArrayList<Class> classes = new ArrayList<>();
        for (File directory : dirs) {
            classes.addAll(findClasses(directory, packageName));
        }
        return classes;
    }

    /**
     * 通过目录来递归查找此目录下所有Class
     */
    private static List<Class> findClasses(File directory, String packageName) throws ClassNotFoundException {
        List<Class> classes = new ArrayList<>();
        if (!directory.exists()) {
            return classes;
        }
        File[] files = directory.listFiles();
        for (File file : files) {
            if (file.isDirectory()) {
                assert !file.getName().contains(".");
                classes.addAll(findClasses(file, packageName + "." + file.getName()));
            } else if (file.getName().endsWith(".class")) {
                classes.add(Class.forName(packageName + '.' + file.getName().substring(0, file.getName().length() - 6)));
            }
        }
        return classes;
    }

    public static void main(String[] args) throws IOException, ClassNotFoundException {
        //获得controller包下的所有Class对象
        List<Class> classes = getClasses("controller");

        Map<String, Class> handleMap = new HashMap<>();

        //1. 遍历controller包下的所有Class
        //2. 如果有@Controller注解,表明这是业务类,继续向下走
        //3. 获得Class的@RequestMapping注解的value值.
        //4. 遍历Class的所有Method,获得方法的@RequestMapping注解的value值,和Class的value值拼接,得到url.
        //5. url和Class加入map.
        classes.forEach(clazz -> {
            Controller controller = (Controller) clazz.getAnnotation(Controller.class);
            if (controller == null) {
                return;
            }
            RequestMapping classRequestMapping = (RequestMapping) clazz.getAnnotation(RequestMapping.class);
            if (classRequestMapping != null) {
                final String classURI = classRequestMapping.value();

                Method[] methods = clazz.getMethods();
                for (Method method : methods) {
                    RequestMapping methodRequestMapping = method.getAnnotation(RequestMapping.class);
                    if (methodRequestMapping != null) {
                        String methodURI = classURI + methodRequestMapping.value();
                        handleMap.put(methodURI, clazz);
                    }
                }
            }


        });

        handleMap.forEach((s, clazz) -> System.out.println(s + " : " + clazz));


    }


}

/main/login : class controller.MainController
/main/logout : class controller.MainController

```

---
## 参考文档
[Java注解](http://www.trinea.cn/android/java-annotation-android-open-source-analysis/)