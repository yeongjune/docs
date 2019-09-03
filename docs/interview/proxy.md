# JAVA 动态代理详解

> 动态代理，听上去很高大上的技术，在Java里应用广泛，尤其是在Hibernate和Spring这两种框架里，在AOP，权限控制，事务管理等方面都有动态代理的实现。JDK本身有实现动态代理技术，但是略有限制，即被代理的类必须实现某个接口，否则无法使用JDK自带的动态代理，因此，如果不满足条件，就只能使用另一种更加灵活，功能更加强大的动态代理技术—— CGLIB。Spring里会自动在JDK的代理和CGLIB之间切换，同时我们也可以强制Spring使用CGLIB。下面我们就动态代理方面的知识点从头至尾依次介绍一下。


我们先来看一个例子：
新建一个接口，UserService.java, 只有一个方法add()。
package com.adam.java.basic;
 
public interface UserService {
	public abstract void add();
}

建一个该接口的实现类UserServiceImpl.java
```
package com.adam.java.basic;
public class UserServiceImpl implements UserService {
	@Override
	public void add() {
		System.out.println("----- add -----");
	}
}

```

建一个代理处理类MyInvocationHandler.java
```
package com.adam.java.basic;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
 
public class MyInvocationHandler implements InvocationHandler {
 
	private Object target;
 
	public MyInvocationHandler(Object target) {
		super();
		this.target = target;
	}
 
	public Object getProxy() {
		return Proxy.newProxyInstance(Thread.currentThread()
				.getContextClassLoader(), target.getClass().getInterfaces(),
				this);
	}
 
	@Override
	public Object invoke(Object proxy, Method method, Object[] args)
			throws Throwable {
		System.out.println("----- before -----");
		Object result = method.invoke(target, args);
		System.out.println("----- after -----");
		return result;
	}
}
```
测试类
```
package com.adam.java.basic;
public class DynamicProxyTest {
 
	public static void main(String[] args) {
		UserService userService = new UserServiceImpl();
		MyInvocationHandler invocationHandler = new MyInvocationHandler(
				userService);
 
		UserService proxy = (UserService) invocationHandler.getProxy();
		proxy.add();
	}
}
```

执行测试类，得到如下输出：
----- before -----
----- add -----
----- after -----
到这里，我们应该会想到点问题：
1. 这个代理对象是由谁且怎么生成的？
2. invoke方法是怎么调用的？
3. invoke和add方法有什么对应关系？
4. 生成的代理对象是什么样子的？
带着这些问题，我们看一下源码。首先，我们的入口便是上面测试类里的getProxy()方法，我们跟进去，看看这个方法：

```
public Object getProxy() {
	return Proxy.newProxyInstance(Thread.currentThread()
			.getContextClassLoader(), target.getClass().getInterfaces(),this);
}

```
也就是说，JDK的动态代理，是通过一个叫Proxy的类来实现的，我们继续跟进去，看看Proxy类的newProxyInstance()方法。先来看看JDK的注释：

```
/**
     * Returns an instance of a proxy class for the specified interfaces
     * that dispatches method invocations to the specified invocation
     * handler.
     *
     * <p>{@code Proxy.newProxyInstance} throws
     * {@code IllegalArgumentException} for the same reasons that
     * {@code Proxy.getProxyClass} does.
     *
     * @param   loader the class loader to define the proxy class
     * @param   interfaces the list of interfaces for the proxy class
     *          to implement
     * @param   h the invocation handler to dispatch method invocations to
     * @return  a proxy instance with the specified invocation handler of a
     *          proxy class that is defined by the specified class loader
     *          and that implements the specified interfaces

```
根据JDK注释我们得知，newProxyInstance方法最终将返回一个实现了指定接口的类的实例，其三个参数分别是：ClassLoader，指定的接口及我们自己定义的InvocationHandler类。我摘几条关键的代码出来，看看这个代理类的实例对象到底是怎么生成的。

```
Class<?> cl = getProxyClass0(loader, intfs);
...
final Constructor<?> cons = cl.getConstructor(constructorParams);
...
return cons.newInstance(new Object[]{h});

```
有兴趣的同学可以自己看看JDK的源码，当前我用的版本是JDK1.8.25，每个版本实现方式可能会不一样，但基本一致，请研究源码的同学注意这一点。上面的代码表明，首先通过getProxyClass获得这个代理类，然后通过c1.getConstructor()拿到构造函数，最后一步，通过cons.newInstance返回这个新的代理类的一个实例，注意：调用newInstance的时候，传入的参数为h，即我们自己定义好的InvocationHandler类，先记着这一步，后面我们就知道这里这样做的原因。
其实这三条代码，核心就是这个getProxyClass方法，另外两行代码是Java反射的应用，和我们当前的兴趣点没什么关系，所以我们继续研究这个getProxyClass方法。这个方法，注释很简单，如下：
```
/*
         * Look up or generate the designated proxy class.
         */
        Class<?> cl = getProxyClass0(loader, intfs);
```
就是生成这个关键的代理类，我们跟进去看一下。
```
private static Class<?> getProxyClass0(ClassLoader loader,
                                           Class<?>... interfaces) {
        if (interfaces.length > 65535) {
            throw new IllegalArgumentException("interface limit exceeded");
        }
 
        // If the proxy class defined by the given loader implementing
        // the given interfaces exists, this will simply return the cached copy;
        // otherwise, it will create the proxy class via the ProxyClassFactory
        return proxyClassCache.get(loader, interfaces);
    }
```
这里用到了缓存，先从缓存里查一下，如果存在，直接返回，不存在就新创建。在这个get方法里，我们看到了如下代码：
Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));
此处提到了apply()，是Proxy类的内部类ProxyClassFactory实现其接口的一个方法，具体实现如下：
```
public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {
 
            Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
            for (Class<?> intf : interfaces) {
                /*
                 * Verify that the class loader resolves the name of this
                 * interface to the same Class object.
                 */
                Class<?> interfaceClass = null;
                try {
                    interfaceClass = Class.forName(intf.getName(), false, loader);
                } catch (ClassNotFoundException e) {
                }
                if (interfaceClass != intf) {
                    throw new IllegalArgumentException(
                        intf + " is not visible from class loader");
                }...
```
看到Class.forName()的时候，我想大多数人会笑了，终于看到熟悉的方法了，没错！这个地方就是要加载指定的接口，既然是生成类，那就要有对应的class字节码，我们继续往下看：
```

/*
 * Generate the specified proxy class.
 */
   byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
   proxyName, interfaces, accessFlags);
    try {
          return defineClass0(loader, proxyName,
          proxyClassFile, 0, proxyClassFile.length);
```
这段代码就是利用ProxyGenerator为我们生成了最终代理类的字节码文件，即getProxyClass0()方法的最终返回值。所以让我们回顾一下最初的四个问题： 
1. 这个代理对象是由谁且怎么生成的？ 

2. invoke方法是怎么调用的？  

3. invoke和add方法有什么对应关系？ 

4. 生成的代理对象是什么样子的？   

对于第一个问题，我想答案已经清楚了，我再屡一下思路：由Proxy类的getProxyClass0()方法生成目标代理类，然后拿到该类的构造方法，最后通过反射的newInstance方法，产生代理类的实例对象。
接下来，我们看看其他的三个方法，我想先从第四个入手，因为有了上面的生成字节码的代码，那我们可以模仿这一步，自己生成字节码文件看看，所以，我用如下代码，生成了这个最终的代理类。
```
package com.adam.java.basic;
 
import java.io.FileOutputStream;
import java.io.IOException;
import sun.misc.ProxyGenerator;
 
public class DynamicProxyTest {
 
	public static void main(String[] args) {
		UserService userService = new UserServiceImpl();
		MyInvocationHandler invocationHandler = new MyInvocationHandler(
				userService);
 
		UserService proxy = (UserService) invocationHandler.getProxy();
		proxy.add();
		
		String path = "C:/$Proxy0.class";
		byte[] classFile = ProxyGenerator.generateProxyClass("$Proxy0",
				UserServiceImpl.class.getInterfaces());
		FileOutputStream out = null;
 
		try {
			out = new FileOutputStream(path);
			out.write(classFile);
			out.flush();
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			try {
				out.close();
			} catch (IOException e) {
				e.printStackTrace();
			}
		}
	}
}
```
上面测试方法里的proxy.add()，此处的add()方法，就已经不是原始的UserService里的add()方法了，而是新生成的代理类的add()方法，我们将生成的$Proxy0.class文件用jd-gui打开，我去掉了一些代码，add()方法如下：
```
public final void add()
    throws 
  {
    try
    {
      this.h.invoke(this, m3, null);
      return;
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }
```
核心就在于this.h.invoke(this. m3, null);此处的h是啥呢？我们看看这个类的类名：
public final class $Proxy0 extends Proxy implements UserService

不难发现，新生成的这个类，继承了Proxy类实现了UserService这个方法，而这个UserService就是我们指定的接口，所以，这里我们基本可以断定，JDK的动态代理，生成的新代理类就是继承了Proxy基类，实现了传入的接口的类。那这个h到底是啥呢？我们再看看这个新代理类，看看构造函数：
```
public $Proxy0(InvocationHandler paramInvocationHandler)
    throws 
  {
    super(paramInvocationHandler);
  }
```
构造函数里传入了一个InvocationHandler类型的参数，看到这里，我们就应该想到之前的一行代码：
return cons.newInstance(new Object[]{h}); 

这是newInstance方法的最后一句，传入的h，就是这里用到的h，也就是我们最初自己定义的MyInvocationHandler类的实例。所以，我们发现，其实最后调用的add()方法，其实调用的是MyInvocationHandler的invoke()方法。我们再来看一下这个方法，找一下m3的含义，继续看代理类的源码：
```
static
  {
    try
    {
      m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] { Class.forName("java.lang.Object") });
      m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
      m3 = Class.forName("com.adam.java.basic.UserService").getMethod("add", new Class[0]);
      m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
      return;
    }
```
惊喜的发现，原来这个m3，就是原接口的add()方法，看到这里，还有什么不明白的呢？我想2,3,4问题都应该迎刃而解了吧？我们继续，看看原始MyInvocationHandler里的invoke()方法：
```
	@Override
	public Object invoke(Object proxy, Method method, Object[] args)
			throws Throwable {
		System.out.println("----- before -----");
		Object result = method.invoke(target, args);
		System.out.println("----- after -----");
		return result;
	}
```
m3就是将要传入的method，所以，为什么先输出before，后输出after，到这里是不是全明白了呢？这，就是JDK的动态代理整个过程，不难吧？
最后，我稍微总结一下JDK动态代理的操作过程：

1. 定义一个接口，该接口里有需要实现的方法，并且编写实际的实现类。

2. 定义一个InvocationHandler类，实现InvocationHandler接口，重写invoke()方法，且添加getProxy()方法。

总结一下动态代理实现过程：

1. 通过getProxyClass0()生成代理类。

2. 通过Proxy.newProxyInstance()生成代理类的实例对象，创建对象时传入InvocationHandler类型的实例。

3. 调用新实例的方法，即此例中的add()，即原InvocationHandler类中的invoke()方法。
————————————————
原文链接：https://blog.csdn.net/zhangerqing/article/details/42504281
