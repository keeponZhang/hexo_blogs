---
title: Groovy 学习笔记
date: 2016-05-18 15:28:53
tags:
---
## 基本概念
Groovy是一中运行在jvm上的动态语言
Groovy注释标记和Java一样，支持//或者/**/
Groovy语句可以不用分号结尾。Groovy为了尽量减少代码的输入，确实煞费苦心
Groovy中支持动态类型，即定义变量的时候可以不指定其类型。Groovy中，变量定义可以使用关键字def。注意，虽然def不是必须的，但是为了代码清晰，建议还是使用def关键字

    def variable1 = 1   //可以不使用分号结尾 
    def  int x = 1  //变量定义时，也可以直接指定类型

当变量没有def等任何定义时，该变量全局有效。类似于javascript中的不用var定义的变量。


函数定义时，参数的类型也可以不指定。比如

    String testFunction(arg1,arg2){//无需指定参数类型
      ...
    }

除了变量定义可以不指定类型外，Groovy中函数的返回值也可以是无类型的。比如：
**无类型的函数定义，必须使用def关键字**

    def  nonReturnTypeFunc(){
        last_line   //最后一行代码的执行结果就是本函数的返回值
    }
    
如果指定了函数返回类型，则可不必加def关键字来定义函数

    String getString(){
       return"I am a string"
    }

其实，所谓的无返回类型的函数，我估计内部都是按返回Object类型来处理的。毕竟，Groovy是基于Java的，而且最终会转成Java Code运行在JVM上

### 字符串

1  单引号''中的内容严格对应Java中的String，不对$符号进行转义  
   
    defsingleQuote='I am $ dolloar'  //输出就是I am $ dolloar

2  双引号""的内容则和脚本语言的处理有点像，如果字符中有$号的话，则它会$表达式先求值。  

    defdoubleQuoteWithoutDollar = "I am one dollar" //输出 I am one dollar  
    def x = 1  
    defdoubleQuoteWithDollar = "I am $x dolloar" //输出I am 1 dolloar  
   
3 三个引号'''xxx'''中的字符串支持随意换行 比如  
   
     defmultieLines = ''' begin  
          line  1  
          line  2  
          end '''

最后，除了每行代码不用加分号外，Groovy中函数调用的时候还可以不加括号。比如：  

    println("test") ---> println"test"  

调用函数要不要带括号，我个人意见是如果这个函数是Groovy API或者Gradle API中比较常用的，比如println，就可以不带括号。否则还是带括号。


## Groovy中的数据类型
### 基本数据类型
作为动态语言，Groovy世界中的所有事物都是对象。所以，int，boolean这些Java中的基本数据类型，在Groovy代码中其实对应的是它们的包装数据类型。比如int对应为Integer，boolean对应为Boolean。

### 闭包

* 闭包中最后一行语句，表示该闭包的返回值，不论该语句是否冠名return关键字


        def aClosure = {//闭包是一段代码，所以需要用花括号括起来..
        Stringparam1, int param2 ->  //这个箭头很关键。箭头前面是参数定义，箭头后面是代码
        println"this is code" //这是代码，最后一句是返回值，
        //也可以使用return，和Groovy中普通函数一样
        }
    
闭包的调用

    aClosure("this is string", 100)  
    


 * 调用闭包的方法等于创建一个闭包实例。对于相同闭包创建出来的不同实例，他们的对象是不同的
 
    

        v1 = c()  
        v2 = c()  
        assert v1 != v2  


 *  如果闭包没定义参数的话，则隐含有一个参数，这个参数名字叫it。

比如：
    
    def greeting = { "Hello, $it!" }
    
    assert greeting('Patrick') == 'Hello, Patrick!'
    等同于：
    def greeting = { it -> "Hello, $it!"}
    assert greeting('Patrick') == 'Hello, Patrick!'
    但是，如果在闭包定义时，采用下面这种写法，则表示闭包没有参数！
    def noParamClosure = { -> true }
    这个时候，我们就不能给noParamClosure传参数了！
    noParamClosure ("test")  <==报错喔！
    
 * 省略圆括号
    
Groovy中，当函数的最后一个参数是闭包的话，在调用该函数的时候可以把闭包拿到圆括号的外面
比如  

    def testClosure(int a1,String b1, Closure closure){  
         closure() //调用闭包  
    }  
    //那么调用的时候，就可以免括号！  
    testClosure (4, "test"）{  
       println"i am in closure"  
    } 
    
如果函数中只有一个参数，且该参数是闭包类型的话，在调用该函数的时候可以省略圆括号

    doLast{
        println "Hello Word"
    }
    //等同于
    doLast({
        println "Hello Word"
    })
    
**省略圆括号使得代码简洁，看起来更像脚本语言**
    
## Groovy脚步的运行原理

Groovy和java一样，都是运行于jvm上的语言。所以Groovy语言在编译时，会被编译成jvm识别的classs文件。
        
![](http://img.blog.csdn.net/20150905192824392?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

参考  [http://blog.csdn.net/innost/article/details/48228651](http://blog.csdn.net/innost/article/details/48228651)
