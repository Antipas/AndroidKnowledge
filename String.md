String
===

```
String s1 = “abc”; 
String s2 = “abc”; 
System.out.println(s1 == s2); 
```

```
String s1 = new String(“abc”); 
String s2 = new String(“abc”); 
System.out.println(s1 == s2); 
```

```
String s1 = “abc”; 
String s2 = “a”; 
String s3 = “bc”; 
String s4 = s2 + s3; 
System.out.println(s1 == s4); 
```


```
String s1 = “abc”; 
final String s2 = “a”; 
final String s3 = “bc”; 
String s4 = s2 + s3; 
System.out.println(s1 == s4); 
```


```
String s = new String(“abc”); 
String s1 = “abc”; 
String s2 = new String(“abc”); 
System.out.println(s == s1.intern()); 
System.out.println(s == s2.intern()); 
System.out.println(s1 == s2.intern()); 
```

##常量概念
像 static final 或 final 修饰的变量适用于常量替换，它们是在方法区里的常量池中的


##intern
intern用来返回常量池中的某字符串，如果常量池中已经存在该字符串，则直接返回常量池中该对象的引用。否则，在常量池中加入该对象，然后 返回引用

JDK1.7之后intern()如果在常量池里找不到，不会复制字符串过去而是生成对原字符串的引用


[原文分析](https://blog.csdn.net/soonfly/article/details/70147205)