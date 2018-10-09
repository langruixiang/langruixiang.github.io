---
layout: post
title: Java动态代理
categories: [Java]
description: 基于JDK实现动态代理
keywords: Java, AOP
---

# 具体场景

* 为了使代理类和被代理类对第三方有相同的函数，代理类和被代理类一般实现一个公共的interface，该interface定义如下

```   
public interface Calculator {
	public Integer add(Integer num1, Integer num2);
	public Integer minus(Integer num1, Integer num2);
}
```
    
* 被代理类定义如下

```
public class CalculatorImpl implements Calculator {

	@Override
	public Integer add(Integer num1, Integer num2) {
		int ret = num1 + num2;
		System.out.println("in calculatorImpl, res: " + ret);
		return ret;
	}
	
	@Override
	public Integer minus(Integer num1, Integer num2) {
		int ret = num1 - num2;
		System.out.println("int calculatorImpl, res: " + ret);
		return ret;
	}

}
```
    
* 代理需求：在add函数和minus函数调用前后分别输出before invocation和after invocation字样

# 静态代理解决方案

代码如下：简单直接，无需赘言，如果calculator里边不仅有add和minus，还有divide，product,log,sin...呢，呵呵哒

    public class StaticCalculatorProxy implements Calculator {
    	Calculator obj;
    	
    	public StaticCalculatorProxy(Calculator obj) {
    		this.obj = obj;	
    	}
    
    	@Override
    	public Integer add(Integer num1, Integer num2) {
    		System.out.println("in StaticCalculatorProxy, before invocation");
    		Integer ret = obj.add(num1, num2);
    		System.out.println("in StaticCalculatorProxy, after invocation");
    		return ret;
    	}
    
    	@Override
    	public Integer minus(Integer num1, Integer num2) {
    		System.out.println("in StaticCalculatorProxy, before invocation");
    		Integer ret = obj.minus(num1, num2);
    		System.out.println("in StaticCalculatorProxy, after invocation");
    		return ret;
    	}
    
    }
# 动态代理解决方案

* 首先编写实现InvocationHandler接口的类，用于请求转发，实现如下

```  
public class CalculatorHandler implements InvocationHandler {
	
	private Object obj; //被代理类
	
	public CalculatorHandler(Object obj) {
		this.obj = obj;
	}

	@Override
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		System.out.println("in calculatorhandler, before invocation");
		
		Object ret = method.invoke(obj, args);  //执行被代理类方法
		
		System.out.println("in calculationhandler, after invocation");
		return ret;
	}

}
```
    
* 生成动态代理

```    
CalculatorImpl calculatorImpl = new CalculatorImpl();//被代理类
CalculatorHandler calculatorHandler = new CalculatorHandler(calculatorImpl);
Calculator calculator = (Calculator) Proxy.newProxyInstance(calculatorImpl.getClass().getClassLoader(), calculatorImpl.getClass().getInterfaces(), calculatorHandler);
System.out.println(calculator.add(1,2));
System.out.println(calculator.minus(1, 2));
无论calculator中包含多少函数，动态代理只需实现一次，实际工程中，System.out.println("in calculatorhandler, before invocation")可能是加缓存，打日志等操作
```

# 动态代理如何工作的
为了搞清楚动态代理如何工作，首先看看生成的动态代理的代码是什么，借助[1]中ProxyUtil代码

```
public class ProxyUtils {

    /**
     * Save proxy class to path
     * 
     * @param path path to save proxy class
     * @param proxyClassName name of proxy class
     * @param interfaces interfaces of proxy class
     * @return
     */
    public static boolean saveProxyClass(String path, String proxyClassName, Class[] interfaces) {
        if (proxyClassName == null || path == null) {
            return false;
        }

        // get byte of proxy class
        byte[] classFile = ProxyGenerator.generateProxyClass(proxyClassName, interfaces);
        FileOutputStream out = null;
        try {
            out = new FileOutputStream(path);
            out.write(classFile);
            out.flush();
            return true;
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                out.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return false;
    }
}
```

得到了生成的动态代理代码如下：

```
public final class $Proxy0 extends Proxy
    implements Calculator
{

    public $Proxy0(InvocationHandler invocationhandler)
    {
        super(invocationhandler);
    }

    public final boolean equals(Object obj)
    {
        try
        {
            return ((Boolean)super.h.invoke(this, m1, new Object[] {
                obj
            })).booleanValue();
        }
        catch(Error _ex) { }
        catch(Throwable throwable)
        {
            throw new UndeclaredThrowableException(throwable);
        }
    }

    public final String toString()
    {
        try
        {
            return (String)super.h.invoke(this, m2, null);
        }
        catch(Error _ex) { }
        catch(Throwable throwable)
        {
            throw new UndeclaredThrowableException(throwable);
        }
    }

    public final Integer minus(Integer integer, Integer integer1)
    {
        try
        {
            return (Integer)super.h.invoke(this, m4, new Object[] {
                integer, integer1
            });
        }
        catch(Error _ex) { }
        catch(Throwable throwable)
        {
            throw new UndeclaredThrowableException(throwable);
        }
    }

    public final Integer add(Integer integer, Integer integer1)
    {
        try
        {
            return (Integer)super.h.invoke(this, m3, new Object[] {
                integer, integer1
            });
        }
        catch(Error _ex) { }
        catch(Throwable throwable)
        {
            throw new UndeclaredThrowableException(throwable);
        }
    }

    public final int hashCode()
    {
        try
        {
            return ((Integer)super.h.invoke(this, m0, null)).intValue();
        }
        catch(Error _ex) { }
        catch(Throwable throwable)
        {
            throw new UndeclaredThrowableException(throwable);
        }
    }

    private static Method m1;
    private static Method m2;
    private static Method m4;
    private static Method m3;
    private static Method m0;

    static 
    {
        try
        {
            m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] {
                Class.forName("java.lang.Object")
            });
            m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
            m4 = Class.forName("com.langrx.mq.Calculator").getMethod("minus", new Class[] {
                Class.forName("java.lang.Integer"), Class.forName("java.lang.Integer")
            });
            m3 = Class.forName("com.langrx.mq.Calculator").getMethod("add", new Class[] {
                Class.forName("java.lang.Integer"), Class.forName("java.lang.Integer")
            });
            m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
        }
        catch(NoSuchMethodException nosuchmethodexception)
        {
            throw new NoSuchMethodError(nosuchmethodexception.getMessage());
        }
        catch(ClassNotFoundException classnotfoundexception)
        {
            throw new NoClassDefFoundError(classnotfoundexception.getMessage());
        }
    }
}
```

* 有点长，按照初始化顺序慢慢来分析，首先分析静态代码块：

```
m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] {
                    Class.forName("java.lang.Object")
                });
m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
m4 = Class.forName("com.langrx.mq.Calculator").getMethod("minus", new Class[] {
                    Class.forName("java.lang.Integer"), Class.forName("java.lang.Integer")
                });
m3 = Class.forName("com.langrx.mq.Calculator").getMethod("add", new Class[] {
                    Class.forName("java.lang.Integer"), Class.forName("java.lang.Integer")
                });
m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
```

得到公共interface中的add函数和minus函数对应的Method方法，同事也得到了equals，toString，hashCode三个函数的Method，所以调用代理类的equals，toString，hashCode也是要执行被代理类的方法的，知道这点很有必要


* 构造函数

```
public $Proxy0(InvocationHandler invocationhandler)
{
        super(invocationhandler);
}
```

初始化了内部的InvocationHandler变量，也就是下文的super.h

* 以add为例看一下请求的转发

```
public final Integer add(Integer integer, Integer integer1)
{
    try
    {
        return (Integer)super.h.invoke(this, m3, new Object[] {
            integer, integer1
        });
    }
    catch(Error _ex) { }
    catch(Throwable throwable)
    {
        throw new UndeclaredThrowableException(throwable);
    }
}
```

super.h.invoke就是invocationhandler.invoke就是传入的CalculatorHandler中实现的

```
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	System.out.println("in calculatorhandler, before invocation");
	
	Object ret = method.invoke(obj, args);  //执行被代理类方法
	
	System.out.println("in calculationhandler, after invocation");
	return ret;
}
```

最终执行的就是CalculatorHandler对应的invoke函数

# 总结

```
Calculator calculator = (Calculator) Proxy.newProxyInstance(calculatorImpl.getClass().getClassLoader(), calculatorImpl.getClass().getInterfaces(), calculatorHandler);
生成动态代理的过程步骤如下[2]：

// InvocationHandlerImpl 实现了 InvocationHandler 接口，并能实现方法调用从代理类到委托类的分派转发
// 其内部通常包含指向委托类实例的引用，用于真正执行分派转发过来的方法调用
InvocationHandler handler = new InvocationHandlerImpl(..); 
 
// 通过 Proxy 为包括 Interface 接口在内的一组接口动态创建代理类的类对象
Class clazz = Proxy.getProxyClass(classLoader, new Class[] { Interface.class, ... }); 
 
// 通过反射从生成的类对象获得构造函数对象
Constructor constructor = clazz.getConstructor(new Class[] { InvocationHandler.class }); 
 
// 通过构造函数对象创建动态代理类实例
Interface Proxy = (Interface)constructor.newInstance(new Object[] { handler });
```

Proxy.newProxyInstance帮我们做了2，3，4步，直接返回给我们一个动态代理对象，代理对象最终执行InvocationHandler中invoke函数。顺便强推文章[2]


# References
1. https://github.com/android-cn/android-open-project-demo/tree/master/java-dynamic-proxy
2. https://www.ibm.com/developerworks/cn/java/j-lo-proxy1/index.html

