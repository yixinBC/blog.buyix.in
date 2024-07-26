---
title: 某Java内存马样本分析
date: 2024-07-26 10:38:06
author: yixinBC
tags:
---

最近空白在“今天你RE了吗”里发了一个Java内存马的样本，据说是已经解了两层加密后提取出来的，感觉用到的一些手法挺有意思的，浅浅分析一下。我使用的反编译工具是jadx

该匿名类的package为org.aspectj.weaver.proxy，查了一下是一个springboot的组件

![混淆后的变量名与方法名](pic1.png)

对类内的变量和函数进行名称混淆，稍稍加大了分析难度

![反射调用特征明显](pic2.png)

大量使用反射调用来访问方法

m45lllIlIllII这个方法被大量调用，其作用暂时未知。

![大量调用m45lllIlIllII](pic3.png)

跟进发现其调用了m46

![m45](pic4.png)
![m46](pic5.png)

先看被大量调用的m57，根据其大致功能，可以将其重命名为decrypt_name

![m57](pic6.png)

难点在于`String temp_str = toString(append(new StringBuilder(), getMethodName(getStackTrace(new RuntimeException())[1])));`
中`temp_str`的取值。写个Java代码测一下：

```java
import java.lang.reflect.Method;

/**
 * test1
 */
public class test1 {
    public static /* synthetic */ StackTraceElement[] getStackTrace(Object obj) throws Exception {
        Method method = RuntimeException.class.getMethod("getStackTrace", new Class[0]);
        method.setAccessible(true);
        return (StackTraceElement[]) method.invoke(obj, new Object[0]);
    }

    public static /* synthetic */ String getMethodName(Object obj) throws Exception {
        Method method = StackTraceElement.class.getMethod("getMethodName", new Class[0]);
        method.setAccessible(true);
        return (String) method.invoke(obj, new Object[0]);
    }

    public static /* synthetic */ StringBuilder append(Object obj, String str) throws Exception {
        Method method = StringBuilder.class.getMethod("append", String.class);
        method.setAccessible(true);
        return (StringBuilder) method.invoke(obj, str);
    }

    public static /* synthetic */ String toString(Object obj) throws Exception {
        Method method = StringBuilder.class.getMethod("toString", new Class[0]);
        method.setAccessible(true);
        return (String) method.invoke(obj, new Object[0]);
    }

    public static void main(String[] args) {
        main1(args);
    }

    public static void main1(String[] args) {
        try {
            StackTraceElement[] stackTrace = getStackTrace(new RuntimeException());
            StringBuilder stringBuilder = new StringBuilder();
            for (StackTraceElement stackTraceElement : stackTrace) {
                append(stringBuilder, getMethodName(stackTraceElement));
                append(stringBuilder, "\n");
            }
            System.out.println(toString(stringBuilder));
            String temp_str = toString(
                    append(new StringBuilder(), getMethodName(getStackTrace(new RuntimeException())[1])));
            System.out.println(temp_str);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

发现temp_str是调用decrypt_name函数的那个方法的方法名。也就是说，这个函数实现了不同方法调用相同函数，key不同。如此，我们遍历一下class文件里的方法名。

这里可以直接用kaitaistruct库解析class文件提取方法名，发现方法名里的不可见字符其实就是\n。我们写出如下python解密脚本：

```python
def decrypt_name(enc, method_name):
    length = len(method_name) - 1
    length2 = len(enc) - 1
    temp = ["" for i in range(length2 + 1)]
    while length2 >= 0:
        temp[length2] = chr((ord("k") ^ ord(enc[length2])) ^ ord(method_name[length]))
        if length2 - 1 < 0:
            break
        temp[length2 - 1] = chr(
            (ord("P") ^ ord(enc[length2])) ^ ord(method_name[length])
        )
        length -= 1
        if length < 0:
            length = len(method_name) - 1
        length2 -= 1
    return "".join(temp)
```

接下去就是苦力活，一个个解密出来，重命名啥的。

jadx反编译出来有问题，换用JEB。

![另一处加密函数](pic7.png)

对于method_list_enc中的项进行DES解密，结果存入method_list中，这两个变量就是那两个95项的字符串数组。

后面的工作纯纯帕鲁，从生成method_list_enc的函数中，先调用decrypt_name解密出来method_list_enc中的每一项，然后使用不同的参数进行des解密每一项，最后从method_list中拿到要反射调用的函数名。最后一个个代回去，就可以恢复代码的逻辑。

后续🍁✌解完了这一层，结果发现这一层只是gzip解压了一个加密过的class文件，后续内存马逻辑在另一个class里

还好后续class用的加密思路一脉相承，但是解出来是一个RSA🥲。有点狠

总结：这个Java内存马的难点就在于其大量使用“动态“的不同的key，使得几乎无法进行批量的解混淆工作。对于RuntimeException的调用栈的使用配合方法名的不可打印字符混淆，是阻碍解混淆工作的一大障碍，后续不同的method使用的DES密钥不同，由下标控制，是解混淆工作的第二重障碍。通过对这个内存马样本的浅薄分析，我发现增加混淆过程与程序执行流的耦合程度，可以较好地对抗批量解混淆的自动化脚本。
