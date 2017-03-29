### Java静态导入static import
import static这个特性在JDK1.5首次出现，一般称为静态导入，用法如下：

```java
import static package.identifier.ClassName.*
```
使用了上面的import static后，ClassName类型的静态成员(包括静态成员变量和静态方法)就被引入到当前命名空间，这样的话在调用ClassName类型的静态成员变量、方法时可以像调用本类内定义的成员一样，而不用这样调用：

```java
ClassName.staticMember
ClassName.staticMethod()
```
可以看一个典型的例子

```java
import static com.example.ClassWithStaticMethod.*;

class StaticImportTest {

    public static void main(String[] args) {
        staticMethod();
    }
}
```

```java
class ClassWithStaticMethod {

    private String string = new String("From ClassWithStaticMethod");

    static void staticMethod() {
        System.out.print(new ClassWithStaticMethod().string);
    }
}
```
StaticImportTest运行后会输出`From ClassWithStaticMethod`。   
**值得注意的一点是ClassWithStaticMethod可以是跟StaticImportTest放在同一文件，也可以是其他文件。**

静态导入用在Enum上可以简化对Enum成员的使用，这也是《Effctive Java》推荐的用法，具体如下：

```java
import static com.example.EnumSetTest.Fruit.*;

class EnumSetTest {

    enum Fruit {
        Apple, Orange, Banana, Tomato
    }

    public static void test() {
        //如果没有 import static com.example.EnumSetTest.Fruit.* ，
        //编译器会报错 Cannot resolve symbol xxx
        EnumSet<Fruit> set = EnumSet.of(Apple, Orange);
        System.out.print(set.contains(Banana));
        System.out.print(set.contains(Fruit.Tomato));
    }

    public static void main(String[] args) {
        test();
    }
}
```
可以看到在使用Fruit的enum成员时可以调用，不需要用`Fruit.XXX`这样的形式。这是因为**enum类型在编译后会将enum成员定义成静态成员变量**，通过反编译EnumSetTest$Fruit.class文件可以验证：

```java
final class com.example.EnumSetTest$Fruit extends java.lang.Enum<com.example.EnumSetTest$Fruit>{
    public static final com.example.EnumSetTest$Fruit Apple;
    public static final com.example.EnumSetTest$Fruit Orange;
    public static final com.example.EnumSetTest$Fruit Banana;
    public static final com.example.EnumSetTest$Fruit Tomato;
    public static com.example.EnumSetTest$Fruit[]values();
    public static com.example.EnumSetTest$Fruit valueOf(java.lang.String);
    static {};
}
```
