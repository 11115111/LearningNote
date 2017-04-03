今天翻《Think in Java》复习方法的重写Override与重载Overload，想起方法不能通过抛出不同的异常实现重载，例如下面这样不能通过编译：

```java
class BlindException extends Exception {}

class IllEyeException extends Exception {}

abstract class Human {
	//编译器会报错：see()已经在Human中定义
	public abstract void see() throws IllEyeException;

	public abstract void see() throws BlindException;

}
```
同时，重写方法不能抛出与被重写方法**层次不同**的已检查异常(checkedException)。*层次不同*的含义比较抽象，我的理解是：`throws SomeException`声明指定了方法能抛出的已检异常必须是SomeException或SomeException的子类，如果子类重写方法的`throws AnotherException`声明中AnotherException是SomeException的超类或者是与SomeException无关的其他类，那就打破了父类方法`throws`声明的层次。还是晕？结合下面代码来理解：

```java
public class MethodClashTest {
   public class MethodClashTest {
    public static void main(String[] args) {
        Human human = new Man();
        try {
            //编译器只知道Human的see()方法抛出IllEyeException，
            //假如Man中see()可以声明为throws Exception并且抛出了不同于IllEyeException的已检异常AnotherException，系统将捕捉不到这个异常。
            //这很明显违背了异常机制的设计初衷。
            human.see();
        } catch (IllEyeException e) {
            e.printStackTrace();
        }

    }
}

class Human {
    public void see() throws IllEyeException {
    }
    
}

class Man extends Human {
	 //编译器报错：与Human的see()冲突，被重写方法不抛出Exception
    //@Override			  
   // public void see() throws Exception {
   //
   //}
   
   //编译器报错：与Human的see()冲突，被重写方法不抛出BlindException
   //
    //@Override			  
   // public void see() throws BlindException {
   //
   //}
   
    //有效代码，不抛出异常。
    @Override			  
    public void see() {
    }
}
```
这里要特别指出子类重写方法可以选择不抛出异常，因为这样没有改变父类方法抛出异常的层次，**throws声明实际上隐式包含了“不抛出异常”**！

有了上面的基础，我想到一个问题：如果子类继承的父类有方法see()，又实现了一个接口也声明了方法see()，并且两个see()方法都有throws声明，那么可能会引起冲突。就像这样：

```java
public class MethodClashTest {
    public static void main(String[] args) {

    }

    interface SightedCreature {

        void see() throws BlindException;
    }

    static class BlindException extends Exception {
    }

    static class IllEyeException extends Exception {
    }

    static class Human {
        public void see() throws IllEyeException {
        }

    }

    static class SightedHuman extends Human implements SightedCreature {
		 //编译器报错：与SightedCreature的see()冲突，被重写方法不抛出IllEyeException
        @Override
        public void see() throws IllEyeException  {
        }
    }

}
```
果然，在类SightedHuman实现方法see()时编译器报错。想彻底理解这个编译错误，我觉得首先需要清楚方法的继承和接口实现是怎么回事，所以接下来做个简单的回顾。

```
public class MethodClashTest {
    public static void main(String[] args) {
        new SightedHuman().see();
    }

    interface SightedCreature {
        void see();
    }

    static class Human {
        public void see(){
            System.out.println("Human seeing");
        }
    }

    static class SightedHuman extends Human implements SightedCreature {}

}
```
定义一个SightedCreature接口，就是有视力的动物，既然有视力那自然就要能“看”，所以给SightedCreature加上`see()`方法。同理，再定义一个Human类，也加上`see()`方法。最后定义继承自Human类的SightedHuman类，并且实现SightedCreature接口。（实际上我们应该避免像这样定义功能重合的类和接口，这里的设计只是为了方便说明问题）。   
这段代码能正常编译并输出`Human seeing`，说明SightedHuman类**继承**了Human类的`see()`方法，并且这个**继承**下来的`see()`方法能够匹配接口SightedCreature声明的`see()`方法，也就是方法签名相同。   
   
上面的代码涉及继承和实现，只要理解了方法的继承和实现就不难分析。我们来给上面代码加点料：

```java
public class MethodClashTest {
    public static void main(String[] args) {
        new SightedHuman().see();
    }

    interface SightedCreature {
        void see();
    }

    static class Human {
        public void see(boolean withLeftEye, boolean withRightEye){
            System.out.println("Human seeing" + (withLeftEye ? " with left eye" : "")
                    + (withRightEye ? (withLeftEye ? " and" : "")+ " with right eye" : ""));
        }
    }

    static class SightedHuman extends Human implements SightedCreature {
		 //必须实现see()方法否则编译器会报错
        @Override
        public void see() {
            see(true, true);
        }
    }

}

```
这段代码输出`Human seeing with left eye and with right eye`。   
可以看到这里Human的`see(boolean withLeftEye, boolean withRightEye)`方法多了两个参数，SightedHuman必须实现方法`see()`。因为SightedHuman能够在`see()`方法里直接调用`see(true, true)`，可以确定是因为继承了Human的`see`方法，那么很自然，SightedHuman实现`see()`方法就是发生了**重载**。

现在回头看上面编译不通过的代码：

```java
public class MethodClashTest {
    public static void main(String[] args) {

    }

    interface SightedCreature {

        void see() throws BlindException;
    }

    static class BlindException extends Exception {
    }

    static class IllEyeException extends Exception {
    }

    static class Human {
        public void see() throws IllEyeException {
        }

    }

    static class SightedHuman extends Human implements SightedCreature {
		 //编译器报错：与SightedCreature的see()冲突，被重写方法不抛出IllEyeException
        @Override
        public void see() throws IllEyeException  {
        }
    }

}
```
结合刚才的回顾和开头的结论，可以分析出这段代码报错的原因：   
	
* 1. SightedHuman的方法`public void see() throws IllEyeException`优先重写Human的`public void see() throws IllEyeException`；
* 2. SightedCreature的方法`void see() throws BlindException`在SightedHuman中不能被实现，因为`throws BlindException`与SightedHuman中已有`see()`方法的`throws IllEyeException`层次不同；
* 3. SightedCreature的方法`void see() throws BlindException`无法在SightedHuman被重载，因为不能通过throws重载。

基于这几个分析，编译器找不到办法满足接口SightedCreature的要求，只能报错。短短几行代码居然涉及了方法的重载、重写、类继承、接口实现，可见掌握基础知识是多么重要。
