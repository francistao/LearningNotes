#String源码分析
---

从一段代码说起：

```
public void stringTest(){
    String a = "a"+"b"+1;
    String b = "ab1";
    System.out.println(a == b);
}
```

大家猜一猜结果如何？如果你的结论是true。好吧，再来一段代码：

```
public void stringTest(){
    String a = new String("ab1");
    String b = "ab1";
    System.out.println(a == b);
}
```

结果如何呢？正确答案是false。

让我们看看经过编译器编译后的代码如何

```
//第一段代码
public void stringTest() {
    String a = "ab1";
    String b = "ab1";
    System.out.println(a == b);
}
```
```
//第二段代码
public void stringTest() {
    String a1 = new String("ab1");
    String b = "ab1";
    System.out.println(a1 == b);
}
```

也就是说第一段代码经过了编译期优化，原因是编译器发现"a"+"b"+1和"ab1"的效果是一样的，都是不可变量组成。但是为什么他们的内存地址会相同呢？如果你对此还有兴趣，那就一起看看String类的一些重要源码吧。


一 String类

String类被final所修饰，也就是说String对象是不可变量，并发程序最喜欢不可变量了。String类实现了Serializable, Comparable<String>, CharSequence接口。

Comparable接口有compareTo(String s)方法，CharSequence接口有length()，charAt(int index)，subSequence(int start,int end)方法。


二 String属性

String类中包含一个不可变的char数组用来存放字符串，一个int型的变量hash用来存放计算后的哈希值。

```
/** The value is used for character storage. */
private final char value[];

/** Cache the hash code for the string */
private int hash; // Default to 0

/** use serialVersionUID from JDK 1.0.2 for interoperability */
private static final long serialVersionUID = -6849794470754667710L;
```

三 String构造函数

```
//不含参数的构造函数，一般没什么用，因为value是不可变量
public String() {
    this.value = new char[0];
}

//参数为String类型
public String(String original) {
    this.value = original.value;
    this.hash = original.hash;
}

//参数为char数组，使用java.utils包中的Arrays类复制
public String(char value[]) {
    this.value = Arrays.copyOf(value, value.length);
}

//从bytes数组中的offset位置开始，将长度为length的字节，以charsetName格式编码，拷贝到value
public String(byte bytes[], int offset, int length, String charsetName)
        throws UnsupportedEncodingException {
    if (charsetName == null)
        throw new NullPointerException("charsetName");
    checkBounds(bytes, offset, length);
    this.value = StringCoding.decode(charsetName, bytes, offset, length);
}

//调用public String(byte bytes[], int offset, int length, String charsetName)构造函数
public String(byte bytes[], String charsetName)
        throws UnsupportedEncodingException {
    this(bytes, 0, bytes.length, charsetName);
}
```

三 String常用方法

```
boolean equals(Object anObject)

public boolean equals(Object anObject) {
    //如果引用的是同一个对象，返回真
    if (this == anObject) {
        return true;
    }
    //如果不是String类型的数据，返回假
    if (anObject instanceof String) {
        String anotherString = (String) anObject;
        int n = value.length;
        //如果char数组长度不相等，返回假
        if (n == anotherString.value.length) {
            char v1[] = value;
            char v2[] = anotherString.value;
            int i = 0;
            //从后往前单个字符判断，如果有不相等，返回假
            while (n-- != 0) {
                if (v1[i] != v2[i])
                        return false;
                i++;
            }
            //每个字符都相等，返回真
            return true;
        }
    }
    return false;
}
```

equals方法经常用得到，它用来判断两个对象从实际意义上是否相等，String对象判断规则：

内存地址相同，则为真。

如果对象类型不是String类型，则为假。否则继续判断。

如果对象长度不相等，则为假。否则继续判断。

从后往前，判断String类中char数组value的单个字符是否相等，有不相等则为假。如果一直相等直到第一个数，则返回真。

由此可以看出，如果对两个超长的字符串进行比较还是非常费时间的。

```
int compareTo(String anotherString)

public int compareTo(String anotherString) {
    //自身对象字符串长度len1
    int len1 = value.length;
    //被比较对象字符串长度len2
    int len2 = anotherString.value.length;
    //取两个字符串长度的最小值lim
    int lim = Math.min(len1, len2);
    char v1[] = value;
    char v2[] = anotherString.value;

    int k = 0;
    //从value的第一个字符开始到最小长度lim处为止，如果字符不相等，返回自身（对象不相等处字符-被比较对象不相等字符）
    while (k < lim) {
        char c1 = v1[k];
        char c2 = v2[k];
        if (c1 != c2) {
            return c1 - c2;
        }
        k++;
    }
    //如果前面都相等，则返回（自身长度-被比较对象长度）
    return len1 - len2;
}
```

这个方法写的很巧妙，先从0开始判断字符大小。如果两个对象能比较字符的地方比较完了还相等，就直接返回自身长度减被比较对象长度，如果两个字符串长度相等，则返回的是0，巧妙地判断了三种情况。

```
int hashCode()

public int hashCode() {
    int h = hash;
    //如果hash没有被计算过，并且字符串不为空，则进行hashCode计算
    if (h == 0 && value.length > 0) {
        char val[] = value;

        //计算过程
        //s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        //hash赋值
        hash = h;
    }
    return h;
}
```

String类重写了hashCode方法，Object中的hashCode方法是一个Native调用。String类的hash采用多项式计算得来，我们完全可以通过不相同的字符串得出同样的hash，所以两个String对象的hashCode相同，并不代表两个String是一样的。

```
boolean startsWith(String prefix,int toffset)

public boolean startsWith(String prefix, int toffset) {
    char ta[] = value;
    int to = toffset;
    char pa[] = prefix.value;
    int po = 0;
    int pc = prefix.value.length;
    // Note: toffset might be near -1>>>1.
    //如果起始地址小于0或者（起始地址+所比较对象长度）大于自身对象长度，返回假
    if ((toffset < 0) || (toffset > value.length - pc)) {
        return false;
    }
    //从所比较对象的末尾开始比较
    while (--pc >= 0) {
        if (ta[to++] != pa[po++]) {
            return false;
        }
    }
    return true;
}

public boolean startsWith(String prefix) {
    return startsWith(prefix, 0);
}

public boolean endsWith(String suffix) {
    return startsWith(suffix, value.length - suffix.value.length);
}
```

起始比较和末尾比较都是比较经常用得到的方法，例如在判断一个字符串是不是http协议的，或者初步判断一个文件是不是mp3文件，都可以采用这个方法进行比较。

```
String concat(String str)

public String concat(String str) {
    int otherLen = str.length();
    //如果被添加的字符串为空，返回对象本身
    if (otherLen == 0) {
        return this;
    }
    int len = value.length;
    char buf[] = Arrays.copyOf(value, len + otherLen);
    str.getChars(buf, len);
    return new String(buf, true);
}
```

concat方法也是经常用的方法之一，它先判断被添加字符串是否为空来决定要不要创建新的对象。

```
String replace(char oldChar,char newChar)

public String replace(char oldChar, char newChar) {
    //新旧值先对比
    if (oldChar != newChar) {
        int len = value.length;
        int i = -1;
        char[] val = value; /* avoid getfield opcode */

        //找到旧值最开始出现的位置
        while (++i < len) {
            if (val[i] == oldChar) {
                break;
            }
        }
        //从那个位置开始，直到末尾，用新值代替出现的旧值
        if (i < len) {
            char buf[] = new char[len];
            for (int j = 0; j < i; j++) {
                buf[j] = val[j];
            }
            while (i < len) {
                char c = val[i];
                buf[i] = (c == oldChar) ? newChar : c;
                i++;
            }
            return new String(buf, true);
        }
    }
    return this;
}
```

这个方法也有讨巧的地方，例如最开始先找出旧值出现的位置，这样节省了一部分对比的时间。replace(String oldStr,String newStr)方法通过正则表达式来判断。

```
String trim()

public String trim() {
    int len = value.length;
    int st = 0;
    char[] val = value;    /* avoid getfield opcode */

    //找到字符串前段没有空格的位置
    while ((st < len) && (val[st] <= ' ')) {
        st++;
    }
    //找到字符串末尾没有空格的位置
    while ((st < len) && (val[len - 1] <= ' ')) {
        len--;
    }
    //如果前后都没有出现空格，返回字符串本身
    return ((st > 0) || (len < value.length)) ? substring(st, len) : this;
}
```

trim方法用起来也6的飞起

```
String intern()

public native String intern();
```

intern方法是Native调用，它的作用是在方法区中的常量池里通过equals方法寻找等值的对象，如果没有找到则在常量池中开辟一片空间存放字符串并返回该对应String的引用，否则直接返回常量池中已存在String对象的引用。

将引言中第二段代码

```
//String a = new String("ab1");
//改为
String a = new String("ab1").intern();
```

则结果为为真，原因在于a所指向的地址来自于常量池，而b所指向的字符串常量默认会调用这个方法，所以a和b都指向了同一个地址空间。

```
int hash32()

private transient int hash32 = 0;
int hash32() {
    int h = hash32;
    if (0 == h) {
       // harmless data race on hash32 here.
       h = sun.misc.Hashing.murmur3_32(HASHING_SEED, value, 0, value.length);

       // ensure result is not zero to avoid recalcing
       h = (0 != h) ? h : 1;

       hash32 = h;
    }

    return h;
}
```

在JDK1.7中，Hash相关集合类在String类作key的情况下，不再使用hashCode方式离散数据，而是采用hash32方法。这个方法默认使用系统当前时间，String类地址，System类地址等作为因子计算得到hash种子，通过hash种子在经过hash得到32位的int型数值。

```
public int length() {
    return value.length;
}
public String toString() {
    return this;
}
public boolean isEmpty() {
    return value.length == 0;
}
public char charAt(int index) {
    if ((index < 0) || (index >= value.length)) {
        throw new StringIndexOutOfBoundsException(index);
    }
    return value[index];
}
```

以上是一些简单的常用方法。


总结

String对象是不可变类型，返回类型为String的String方法每次返回的都是新的String对象，除了某些方法的某些特定条件返回自身。

String对象的三种比较方式：

==内存比较：直接对比两个引用所指向的内存值，精确简洁直接明了。

equals字符串值比较：比较两个引用所指对象字面值是否相等。

hashCode字符串数值化比较：将字符串数值化。两个引用的hashCode相同，不保证内存一定相同，不保证字面值一定相同。


