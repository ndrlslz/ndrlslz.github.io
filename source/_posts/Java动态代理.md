title: Java动态代理
date: 2016-07-09 22:27:52
tags:
---

动态代理在框架层面大量使用，比如AOP框架, Retrofit等。这里学习下Java的动态代理。

## 静态代理

静态代理是设计模式中的一种，通过对原始对象提供一个代理类，实现在不改变原始类的情况下，改变原始类的方法。

静态代理是在程序运行前就需要写好代理类，确定好代理类和被代理类的关系。缺点就是对于每个被代理类，都需要写个代理类与之对应。

---
## 动态代理

### 概念
与静态代理不同，动态代理的代理类是在运行时创建的，即代理类和被代理类的关系是在运行时才确定的。好处在于可以处理被代理类预先不知道的情况，而且也解决了静态代理的缺点，通过一个类便可以代理多个原始类。

<!-- more -->

### 创建动态代理
**创建方法**
通过`Proxy.newProxyInstance()`便可以创建动态代理类。

```
/**
 * ClassLoader 加载代理类的ClassLoader
 * interfaces 被代理的接口数组,即这个代理类需要实现哪些接口中的方法.
 * InvocationHandler接口 代理类的主要逻辑, 用来代理，改变原始类中的方法.
 */
public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h)
```

然后需要实现`InvocationHandler`接口的`invoke()`方法。
```
/**
 * 调用被代理类的方法的时候,实际上是调用这个invoke()方法.
 *
 * proxy 生成的动态代理类的引用,一般不会调用它的方法, 可以用来通过反射获得动态代理类的方法，注解等信息。
 * method 被代理的方法
 * args 被代理的方法的参数数组
 */
public Object invoke(Object proxy, Method method, Object[] args)
```

**动态代理实例**
这里来实现一个简单的动态代理。
传入一个类的Class对象，返回这个类的动态代理类，其中改变了这个实现类的方法。

```
package proxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class DynamicProxy {
    public static <T> Object proxy(Class<T> clazz) throws IllegalAccessException, InstantiationException {
        T object = clazz.newInstance();

        return Proxy.newProxyInstance(clazz.getClassLoader(),
                clazz.getInterfaces(),
                new InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        System.out.println("start to inject code");
                        Object result = method.invoke(object, args);
                        System.out.println("end to inject code");
                        return result;
                    }
                });
    }
}
```

然后我们来使用这个动态代理。
需要知道，这种创建动态代理方式的缺点在于它只能针对接口来创建代理类。因为创建出来的代理类已经继承于`Proxy`类了，而Java是单继承的，只能实现被代理的接口了。

我们创建一个`PersonInterface`接口和其实现类`Person`。
```
public interface PersonInterface {
    void say();
}

public class Person implements PersonInterface{
    @Override
    public void say() {
        System.out.println("hello");
    }
}
```

现在，我们需要动态改变Person类的`say`方法，利用之前写好的`DynamicProxy`类来创建动态代理类。
```
public static void main(String[] args) throws InstantiationException, IllegalAccessException {
    PersonInterface person = (PersonInterface) DynamicProxy.proxy(Person.class);
    person.say();
}

//start to inject code
//hello
//end to inject code
```

## 思考Retrofit的create方法
下面是Retrofit的`create()`方法的源码。
```
public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if(this.validateEagerly) {
        this.eagerlyValidateMethods(service);
    }

    return Proxy.newProxyInstance(service.getClassLoader(), new Class[]{service}, new InvocationHandler() {
        private final Platform platform = Platform.get();

        public Object invoke(Object proxy, Method method, Object... args) throws Throwable {
            if(method.getDeclaringClass() == Object.class) {
                return method.invoke(this, args);
            } else if(this.platform.isDefaultMethod(method)) {
                return this.platform.invokeDefaultMethod(method, service, proxy, args);
            } else {
                ServiceMethod serviceMethod = Retrofit.this.loadServiceMethod(method);
                OkHttpCall okHttpCall = new OkHttpCall(serviceMethod, args);
                return serviceMethod.callAdapter.adapt(okHttpCall);
            }
        }
    });
}
```

可以看到，它正是使用了Java的动态代理。
通过传入用户的接口，返回针对这个接口的代理类。

之前有一点我没想明白，就是`create()`方法为什么只需要传接口就可以了。因为通常来说，我们是需要对具体的实现类来进行代理的，而传入的接口是无法实例化的。

后来我明白了，这正是Retrofit不同的地方。它不需要对具体的实例类进行代理，因为所有的功能都是通过动态代理类来做了，而传入接口的作用是只需要通过接口得到方法的参数，返回值，注解等信息。然后利用这些信息来创建REST请求，对返回结果反序列化等等。

---
## cglib

### 概念 
上面这种方式的缺点之前提到了，就是只能对接口的方法进行代理。用`cglib`这个库的话可以直接对类进行代理。

cglib底层使用ASM框架，`ASM`是一个java字节码操纵框架，可以直接产生二进制 class 文件。

### 使用方式
通过实现`MethodInterceptor`接口的`intercept()`方法来实现代理功能，这个方法和之前的`invoke`差不多。
```
/**
 * obj 生成的动态代理类的引用
 * method 被代理的方法
 * args 方法参数数组
 * proxy 代理类的方法的引用, 如果要调用被代理类的方法, 请使用proxy.invokeSuper().
 */
public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy)
```

### 创建动态代理实例
写个简单的例子来创建动态代理对象。
```
public class CglibDynamicProxy {

    @SuppressWarnings("unchecked")
    public static <T> T proxy(Class<T> clazz) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(clazz);
        enhancer.setCallback((MethodInterceptor) (o, method, args, methodProxy) -> {
            System.out.println("start to inject code");
            Object result = methodProxy.invokeSuper(o, args);
            System.out.println("end to inject code");
            return result;
        });
        return (T) enhancer.create();
    }
}
```

上面的代码，通过`setSuperclass()`来指定被代理类，因为最后生成的代理类是这个被代理类的子类。
通过`setCallback()`来指定代理的主要操作。
通过`create()`来创建最后的动态代理类。

### 使用动态代理
我们创建个类来使用这个动态代理功能。
```
public class Test {
    public static void main(String[] args) {
        MyClass myClass = CglibDynamicProxy.proxy(MyClass.class);
        myClass.say();
    }
}

class MyClass {
    public void say() {
        System.out.println("hello");
    }
}

//start to inject code
//hello
//end to inject code
```

---
## 参考文档
[Java动态代理](http://www.jasongj.com/design_pattern/dynamic_proxy_cglib/)
[Retrofit](http://www.jianshu.com/p/fb8d21978e38)
