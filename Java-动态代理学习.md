---
title: Java 动态代理学习
date: 2016-05-05 14:03:47
tags:
---

Java中代理分为2种， 静态代理和动态代理。

##  静态代理

![](http://77fzym.com1.z0.glb.clouddn.com/proxy.png)

从图中可以看出，代理对象是客户端和真实对象之间的桥梁。代理对象要实现真实对象的所有公开方法（或者接口），而且当真实对象的接口修改的时候，代理对象也要做修改。而且在构造代理对象的时候，必须要知道真实对象。就是说代理对象和真实对象在编译时就已经绑定了。

## 动态代理

先来看一个简单的demo

```
public interface Subject{      
  public void doSomething();   
}   

public class RealSubject implements Subject{
  public void doSomething(){
    System.out.println( "call doSomething()" );   
  }   
}   

public class ProxyHandler implements InvocationHandler{
  private Object proxied;   
     
  public ProxyHandler( Object proxied ) { 
    this.proxied = proxied;   
  }   
     
  public Object invoke( Object proxy, Method method, Object[] args ) throws Throwable{
    //在转调具体目标对象之前，可以执行一些功能处理

    //转调具体目标对象的方法
    return method.invoke( proxied, args);  
    
    //在转调具体目标对象之后，可以执行一些功能处理
  }    
}
```

如何使用

```
  public static void main( String args[] ){
     RealSubject real = new RealSubject();   
     Subject proxySubject = (Subject)Proxy.newProxyInstance(Subject.class.getClassLoader(), 
     	new Class[]{Subject.class}, 
     	new ProxyHandler(real)
     );
         
     proxySubject.doSomething();
  }   
```

代理类获知或者被告知真实对象的类型、接口列表时，代理可以根据反射构造出真实对象来，进而调用真实对象的相应方法，这就是动态代理。**代理类在定义时不需要知道具体要代理哪个真实对象，我们在运行时告诉代理类这些好了**。



**动态代理的内部实现**

动态代理就是根据代理类实现的接口来动态生成class文件，然后根据这个class文件动态生成代理对象。

生成动态代理类的字节码并且保存到硬盘中：

JDK提供了**sun.misc.ProxyGenerator.generateProxyClass(String proxyName,class[] interfaces)** 底层方法来产生动态代理类的字节码：

下面定义了一个工具类，用来将生成的动态代理类保存到硬盘中：

	import java.io.FileOutputStream;
	import java.io.IOException;
	import java.lang.reflect.Proxy;
	import sun.misc.ProxyGenerator;
	
	public class ProxyUtils {
	/*
	 * 将根据类信息 动态生成的二进制字节码保存到硬盘中，
	 * 默认的是clazz目录下
	     * params :clazz 需要生成动态代理类的类
	     * proxyName : 为动态生成的代理类的名称
	    &nbsp;*/
	public static void generateClassFile(Class clazz,String proxyName){
		//根据类信息和提供的代理类名称，生成字节码
	    byte[] classFile =ProxyGenerator.generateProxyClass(proxyName, clazz.getInterfaces()); 
		String paths = clazz.getResource(".").getPath();
		System.out.println(paths);
		FileOutputStream out = null;  
	    
	    try {
	        //保留到硬盘中
	        out = new FileOutputStream(paths+proxyName+".class");  
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

现在我们想将生成的代理类起名为“ElectricCarProxy”，并保存在硬盘，应该使用以下语句：

	ProxyUtils.generateClassFile(car.getClass(), "ElectricCarProxy");

然后我们反编译生成的ElectricCarProxy.class文件,反编译之后看到的代码是：


**备注：代码中的 Subject接口 就是代理类实现的接口**


	import java.lang.reflect.*;   
	public final class ProxySubject extends Proxy  implements Subject       
	{   
	    private static Method m1;   
	    private static Method m0;   
	    private static Method m3;   
	    private static Method m2;   
	    public ProxySubject(InvocationHandler invocationhandler)   
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
	    public final void doSomething()   
	    {   
	        try  
	        {   
	            super.h.invoke(this, m3, null);   
	            return;   
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
	    static    
	    {   
	        try  
	        {   
	            m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] {   
	                Class.forName("java.lang.Object")   
	            });   
	            m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);   
	            m3 = Class.forName("Subject").getMethod("doSomething", new Class[0]);   
	            m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);   
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

我们可以看到动态生成的ProxySubject 是继承Proxy类的。这就是
**为什么只能通过接口类来生成动态代理，而不是类来实现，因为java不支持多继承**。
通过反编译之后的代码，我们可以看出调用代理类的一个方法时，代理类的方法会调用InvocationHandler的invoke方法，然后invoke方法再调用被代理对象的方法，所以我们在InvocationHandler的invoke方法的方法里做一些其它的处理操作，比如添加日志等等。



参考

[http://www.cnblogs.com/linjiqin/archive/2011/02/18/1957600.html](http://www.cnblogs.com/linjiqin/archive/2011/02/18/1957600.html)

[http://www.cnblogs.com/flyoung2008/archive/2013/08/11/3251148.html](http://www.cnblogs.com/flyoung2008/archive/2013/08/11/3251148.html)

[http://blog.csdn.net/luanlouis/article/details/24589193](http://blog.csdn.net/luanlouis/article/details/24589193)










