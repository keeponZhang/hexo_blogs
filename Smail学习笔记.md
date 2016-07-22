# Smali学习笔记

## 什么是Smali
由于Android开发是基于java语言开发的，然后运行在dalvik虚拟机上面。dalvik和jvm一样，不能识别java代码，只能识别java编译成的中间代码，在android平台上，这个中间代码就是Smali。通过学习Smali指令我们可以了解代码的运行过程。

## Smali分析

我们从一个简单的例子来分析Smali，我写了一个非常简单的demo，

    public class MainActivity extends Activity {

	    Button btn;
	
	    @Override
	    protected void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_main);
	
	        btn=(Button)findViewById(R.id.btn);
	
	        btn.setOnClickListener(new View.OnClickListener() {
	            @Override
	            public void onClick(View view) {
	                Toast.makeText(MainActivity.this,"Hello World",Toast.LENGTH_SHORT).show();
	            }
	        });
	    }
	}

demo非常简单，就是点击一个Button，弹出一个Hello World的Toast。然后通过apktool来反编译这个程序的Apk文件，反编译成功之后，会生成一个和Apk同名的文件夹，在这个文件夹中我们会看到一个名为smali的文件夹，这个smali文件夹中的目录结构和java源文件的目录结构是一样的。顺着目录我们会看到MainActivity.smali 和 MainActivity$1.smali这2个文件。为什么会有MainActivity$1.smali这个文件呢，因为在给btn按钮注册回调事件是一个匿名内部内，在java中每一个内部类编译之后都会单独生成一个文件，内部类文件的命名是用$符号来命名的。 比如：主类名+$+序号。
先打开MainActivity.smali这个文件,文件的内容如下：
	
	.class public Lcom/mj/mjapp/MainActivity;
	.super Landroid/app/Activity;
	.source "MainActivity.java"


	# instance fields
	.field btn:Landroid/widget/Button;


	# direct methods
	.method public constructor <init>()V
	    .locals 0

	    .prologue
	    .line 9
	    invoke-direct {p0}, Landroid/app/Activity;-><init>()V

	    return-void
	.end method


	# virtual methods
	.method protected onCreate(Landroid/os/Bundle;)V
	    .locals 2
	    .param p1, "savedInstanceState"    # Landroid/os/Bundle;

	    .prologue
	    .line 15
	    invoke-super {p0, p1}, Landroid/app/Activity;->onCreate(Landroid/os/Bundle;)V

	    .line 16
	    const/high16 v0, 0x7f030000

	    invoke-virtual {p0, v0}, Lcom/mj/mjapp/MainActivity;->setContentView(I)V

	    .line 18
	    const/high16 v0, 0x7f070000

	    invoke-virtual {p0, v0}, Lcom/mj/mjapp/MainActivity;->findViewById(I)Landroid/view/View;

	    move-result-object v0

	    check-cast v0, Landroid/widget/Button;

	    iput-object v0, p0, Lcom/mj/mjapp/MainActivity;->btn:Landroid/widget/Button;

	    .line 20
	    iget-object v0, p0, Lcom/mj/mjapp/MainActivity;->btn:Landroid/widget/Button;

	    new-instance v1, Lcom/mj/mjapp/MainActivity$1;

	    invoke-direct {v1, p0}, Lcom/mj/mjapp/MainActivity$1;-><init>(Lcom/mj/mjapp/MainActivity;)V

	    invoke-virtual {v0, v1}, Landroid/widget/Button;->setOnClickListener(Landroid/view/View$OnClickListener;)V

	    .line 26
	    return-void
	.end method


最前面的三行是对当前类的介绍信息。当前类是 Lcom/mj/mjapp/MainActivity; 父类是 Landroid/app/Activity; 当前类的源文件是 MainActivity.java。

**smali类型介绍**
 在smali中，数据类型和Android中的一样，只是对应的符号有变化：

> B---byte
> 
> C---char
> 
> D---double
> 
> F---float
> 
> I---int
> 
> J---long
> 
> S---short
> 
> V---void
> 
> Z---boolean
> 
> [XXX---array
> 
> Lxxx/yyy---object 
   
这里解析下最后两项，数组的表示方式是：在基本类型前加上前中括号“[”，例如int数组和float数组分别表示为：[I、[F；对象的表示则以L作为开头，格式是LpackageName/objectName;（注意必须有个分号跟在最后），例如String对象在smali中为：Ljava/lang/String;，其中java/lang对应java.lang包，String就是定义在该包中的一个对象。所以上面用 **Lcom/mj/mjapp/MainActivity;** 来表示对象，而且末尾用 **;** 。
接着继续往下介绍，

	# instance fields
	.field btn:Landroid/widget/Button;

这个代表声明的实例字段，如果是静态字段用 # static fields 来表示，定义字段的格式是： **.field public/private [static] [final] varName:<类型>** 。

继续往下介绍，

	# direct methods
	.method public constructor <init>()V
	    .locals 0

	    .prologue
	    .line 9
	    invoke-direct {p0}, Landroid/app/Activity;-><init>()V

	    return-void
	.end method


函数的定义一般为：

     Func-Name (Para-Type1Para-Type2Para-Type3...)Return-Type

     注意参数与参数之间没有任何分隔符，同样举几个例子就容易明白了：

     1. foo ()V

         没错，这就是void foo()。

     2. foo (III)Z

         这个则是boolean foo(int, int, int)。

     3. foo (Z[I[ILjava/lang/String;J)Ljava/lang/String;

         看出来这是String foo (boolean, int[], int[], String, long) 了吗？

在smali中 方法的类型 分为3种 ： direct methods，virtual methods和static methods。 构造方法或者是private修饰的方法属于direct methods，静态方法属于static methods, 用public或protected修饰的方法就是属于virtual methods。

现在返回到方法本身   **<init>（）**表示的是构造方法。 

* .locals 0 表示的是需要 0个本地寄存器变量 。如果当前方法需要3个本地变量，就是 .locals 3 。

* .prologue 表示方法的开始。

* .line 9 表示当前的源码对应源码的第9行。 一些混淆后的代码有可能是没有 .prologue 和  .line 9的。真是有了这个，我们才可以在异常栈中看到哪个方法的哪一行出现了异常，才可以快速的定位到出现异常的代码。

* invoke-direct {p0}, Landroid/app/Activity;-><init>()V 

> Dalvik VM与JVM的最大的区别之一就是Dalvik VM是基于寄存器的。基于寄存器是什么意思呢？也就是说，在smali里的所有操作都必须经过寄存器来进行：本地寄存器用v开头数字结尾的符号来表示，如v0、v1、v2、...参数寄存器则使用p开头数字结尾的符号来表示，如p0、p1、p2、...特别注意的是，p0不一定是函数中的第一个参数，在非static函数中，p0代指“this”，p1表示函数的第一个参数，p2代表函数中的第二个参数…而在static函数中p0才对应第一个参数（因为Java的static方法中没有this指针）。

这里的 P0代表的是当前的this对象，这行代码表示 调用Landroid/app/Activity;类的<init>()方法，且传递的参数是p0对象。

* return-void
* end method

上面2行表示函数返回void和函数的结束。

接着往下看：

	# virtual methods
	.method protected onCreate(Landroid/os/Bundle;)V
	    .locals 2
	    .param p1, "savedInstanceState"    # Landroid/os/Bundle;

	    .prologue
	    .line 15
	    invoke-super {p0, p1}, Landroid/app/Activity;->onCreate(Landroid/os/Bundle;)V

	    .line 16
	    const/high16 v0, 0x7f030000

	    invoke-virtual {p0, v0}, Lcom/mj/mjapp/MainActivity;->setContentView(I)V

	    .line 18
	    const/high16 v0, 0x7f070000

	    invoke-virtual {p0, v0}, Lcom/mj/mjapp/MainActivity;->findViewById(I)Landroid/view/View;

	    move-result-object v0

	    check-cast v0, Landroid/widget/Button;

	    iput-object v0, p0, Lcom/mj/mjapp/MainActivity;->btn:Landroid/widget/Button;

	    .line 20
	    iget-object v0, p0, Lcom/mj/mjapp/MainActivity;->btn:Landroid/widget/Button;

	    new-instance v1, Lcom/mj/mjapp/MainActivity$1;

	    invoke-direct {v1, p0}, Lcom/mj/mjapp/MainActivity$1;-><init>(Lcom/mj/mjapp/MainActivity;)V

	    invoke-virtual {v0, v1}, Landroid/widget/Button;->setOnClickListener(Landroid/view/View$OnClickListener;)V

	    .line 26
	    return-void
	.end method

当前的方法的名字是onCreate，是个protected方法，返回类型是void， 参数的类型是Landroid/os/Bundle;。

*  .locals 2 表示当前方法需要2个本地寄存器变量，这2个变量不包括形参变量，这里为什么需要2个本地寄存器变量我会在后面介绍。
*  .param p1, "savedInstanceState"    # Landroid/os/Bundle; 在前面说过 p0表示隐藏的参数，P1才表示当前方法的第一个参数，源码中的参数名是 savedInstanceState。
*  invoke-super {p0, p1}, Landroid/app/Activity;->onCreate(Landroid/os/Bundle;)V  invoke-super指令 表示调用父类的onCreate方法，{p0, p1}表示传递的参数。
*  const/high16 v0, 0x7f030000  const指令表示赋值操作，把0x7f030000赋值给v0，其实0x7f030000 对应的值就是 R.layout.activity_main 这个int类型的值。 const/high16 这个地方的high16我暂时不知道是什么意思。如果哪位网友知道的话，请在评论下面回复我。
*  invoke-virtual {p0, v0}, Lcom/mj/mjapp/MainActivity;->setContentView(I)V  调用setContentView() 方法，把上一步中的v0变量当作这次调用的参数传递给setContentView()方法。
*  const/high16 v0, 0x7f070000 继续给v0赋值，新的值会覆盖旧的值。
*  invoke-virtual {p0, v0}, Lcom/mj/mjapp/MainActivity;->findViewById(I)Landroid/view/View; 调用findViewById()方法。这个方法的返回类型是 Landroid/view/View;
*  move-result-object v0
> 在Java代码中调用函数和返回函数结果是一条语句完成的，而在smali里则需要分开来完成，在使用上述指令后，如果调用的函数返回非void，那么还需要用到move-result（返回基本数据类型）和move-result-object（返回对象）指令。

因为 findViewById()方法返回一个View对象，所以需要调用 move-result-object这个指令，再把返回的值赋值给v0变量。我们发现申请的一个变量是可以重复使用的。

*  check-cast v0, Landroid/widget/Button; check-cast指令表示的是 类型转化。因为v0表示的是View类型，需要将其转化成Button类型。
*  iput-object v0, p0, Lcom/mj/mjapp/MainActivity;->btn:Landroid/widget/Button; 
> 一般来说，获取值的指令有：iget、sget、iget-boolean、sget-boolean、iget-object、sget-object等，设置值操作的指令有：iput、sput、iput-boolean、sput-boolean、iput-object、sput-object等。没有“-object”后缀的表示操作的成员变量对象是基本数据类型，带“-object”表示操作的成员变量是对象类型，特别地，boolean类型则使用带“-boolean”的指令操作。iget表示的是获得实例变量的值，sget表示的是活的静态变量的值。同理 iput和sput表示设置实例变量和静态变量的值。iput和iget指令操作的是实例变量，而sput和sget指令操作的是静态变量。所以iput和iget指令会比sput和sget指令多一个参数，多出的这个参数表示当前的操作的实例变量属于哪个对象。多出的那个参数是在iput和iget指令三个参数中的中间的那个参数。

上面指令的意思是把v0变量赋值给p0对象的btn变量。

*  iget-object v0, p0, Lcom/mj/mjapp/MainActivity;->btn:Landroid/widget/Button; 这个是把p0对象的btn变量赋值给 v0变量。
*  new-instance v1, Lcom/mj/mjapp/MainActivity$1; 实例化Lcom/mj/mjapp/MainActivity$1;这个内部类变量，然后赋值给v1这个变量。Lcom/mj/mjapp/MainActivity$1;类里面smali代码我等下会介绍。
*  invoke-direct {v1, p0}, Lcom/mj/mjapp/MainActivity$1;-><init>(Lcom/mj/mjapp/MainActivity;)V 调用内部类的构造方法。
*  invoke-virtual {v0, v1}, Landroid/widget/Button;->setOnClickListener(Landroid/view/View$OnClickListener;)V 调用Button对象的setOnClickListener()方法。

现在我们返回到方法开始处的 .locals 2 的问题，因为该方法中 申请了 v0 和v1 2个本地寄存器变量，所以是 .locals 2。如果把2改成3，那么重新打包运行会得到一个VerifyError错误。
我们已经分析完了MainActivity.smali 这个文件中的smali代码，接着继续分析MainActivity$1.smali中的代码。

MainActivity$1.smali代码如下：

	.class Lcom/mj/mjapp/MainActivity$1;
	.super Ljava/lang/Object;
	.source "MainActivity.java"

	# interfaces
	.implements Landroid/view/View$OnClickListener;


	# annotations
	.annotation system Ldalvik/annotation/EnclosingMethod;
	    value = Lcom/mj/mjapp/MainActivity;->onCreate(Landroid/os/Bundle;)V
	.end annotation

	.annotation system Ldalvik/annotation/InnerClass;
	    accessFlags = 0x0
	    name = null
	.end annotation


	# instance fields
	.field final synthetic this$0:Lcom/mj/mjapp/MainActivity;


	# direct methods
	.method constructor <init>(Lcom/mj/mjapp/MainActivity;)V
	    .locals 0

	    .prologue
	    .line 26
	    iput-object p1, p0, Lcom/mj/mjapp/MainActivity$1;->this$0:Lcom/mj/mjapp/MainActivity;

	    invoke-direct {p0}, Ljava/lang/Object;-><init>()V

	    return-void
	.end method


	# virtual methods
	.method public onClick(Landroid/view/View;)V
	    .locals 3
	    .param p1, "view"    # Landroid/view/View;

	    .prologue
	    .line 29
	    iget-object v0, p0, Lcom/mj/mjapp/MainActivity$1;->this$0:Lcom/mj/mjapp/MainActivity;

	    const-string v1, "Hello world"

	    const/4 v2, 0x0

	    invoke-static {v0, v1, v2}, Landroid/widget/Toast;->makeText(Landroid/content/Context;Ljava/lang/CharSequence;I)Landroid/widget/Toast;

	    move-result-object v0

	    invoke-virtual {v0}, Landroid/widget/Toast;->show()V

	    .line 30
	    return-void
	.end method

接下来我们选重点讲：

	# interfaces
	.implements Landroid/view/View$OnClickListener;

这句代码表示 当前类实现了 Landroid/view/View$OnClickListener;这个接口。

	# annotations
	.annotation system Ldalvik/annotation/EnclosingMethod;
	    value = Lcom/mj/mjapp/MainActivity;->onCreate(Landroid/os/Bundle;)V
	.end annotation

	.annotation system Ldalvik/annotation/InnerClass;
	    accessFlags = 0x0
	    name = null
	.end annotation

上面注解是对当前内部类的一个解释说明，

*  .annotation system Ldalvik/annotation/EnclosingMethod; 这句代码说明当前内部类作用于一个方法。与EnclosingMethod对应的还有 EnclosingClass注解。

*  value = Lcom/mj/mjapp/MainActivity;->onCreate(Landroid/os/Bundle;)V 这句代码表示当前内部类位于Lcom/mj/mjapp/MainActivity;的onCreate方法内。
*  accessFlags = 0x0  accessFlags 访问标志是一个枚举值
  
  该枚举的声明如下：
		
		enum {
		    kDexVisibilityBuild      = 0x00,     /* annotation visibility */
		    kDexVisibilityRuntime    = 0x01,
			kDexVisibilitySystem     = 0x02,
		};

 accessFlags = 0x0 表明它的属性是 Runtime。name为内部类的名称。接着继续往下看

	# instance fields
	.field final synthetic this$0:Lcom/mj/mjapp/MainActivity;

 声明一个外部内的实例对象this$0 ，通过这句代码我们可以看出每一个内部类都有一个指向外部类的引用。而且声明为final，synthetic表示该行代码是编译器生成的。


	# direct methods
	.method constructor <init>(Lcom/mj/mjapp/MainActivity;)V
	    .locals 0

	    .prologue
	    .line 26
	    iput-object p1, p0, Lcom/mj/mjapp/MainActivity$1;->this$0:Lcom/mj/mjapp/MainActivity;

	    invoke-direct {p0}, Ljava/lang/Object;-><init>()V

	    return-void
	.end method

这几行代码表示当前内部内的默认构造方法，在该构造方法中为 this$0 这个变量赋值为p1。这个p1是从哪儿来的？ 我们往回看外部类的onCreate方法，我们看到有这么一行代码
	
> invoke-direct {v1, p0}, Lcom/mj/mjapp/MainActivity$1;-><init>(Lcom/mj/mjapp/MainActivity;)V

通过这行代码可以看出这个参数列表中的p0就是内部类的构造方法中的p1 。

*  invoke-direct {p0}, Ljava/lang/Object;-><init>()V  调用 Ljava/lang/Object;对象的无参构造方法。

接着看 内部类的onClick方法。

*  .locals 3 表示声明了3个本地寄存器变量。
*  iget-object v0, p0, Lcom/mj/mjapp/MainActivity$1;->this$0:Lcom/mj/mjapp/MainActivity; 把this$0这个实例变量赋值给v0这个寄存器本地变量。
*  const-string v1, "Hello world" 把 "Hello world" 这个字符串赋值给 v1这个本地变量。
*  const/4 v2, 0x0 把0x0赋值给v2 这个本地变量。
*  invoke-static {v0, v1, v2}, Landroid/widget/Toast;->makeText(Landroid/content/Context;Ljava/lang/CharSequence;I)Landroid/widget/Toast;  invoke-static指令表示调用Landroid/widget/Toast;这个类的makeText这个静态方法，v0,v1,v2作为三个参数传递给makeText方法。
*  move-result-object v0 把Landroid/widget/Toast;->makeText(Landroid/content/Context;Ljava/lang/CharSequence;I)Landroid/widget/Toast;这个方法的返回值赋值给v0。
*  invoke-virtual {v0}, Landroid/widget/Toast;->show()V 调用Toast的show()方法。

至此 内部类的smali代码也分析完成。
