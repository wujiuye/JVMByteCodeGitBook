# 字节码指令集

![Java艺术](../qrcode/javaskill_qrcode_01.png)

Java虚拟机的指令是由一个字节长度的、代表着某种特定操作含义的数字（称为操作码，Opcode）以及跟随其后的零个或多个代表此操作所需参数（称为操作数，Operand）而构成[^1]。 虽然一条虚拟机指令的操作码只用一个字节存储，Java虚拟机所能支持的指令最多只能有256条，但虚拟机指令很少会随着虚拟机版本的更新而增加新的指令，即便Java版本更新很快。

相信通过前面第二章的学习，我们都已经在枯燥的学习中掌握了class文件结构，相比于学习 class文件结构，本章介绍的字节码指令会增添几分乐趣。在了解每条字节码指令后，通过javap查看Java代码编译后的字节码就能了解一些语法糖的本质实现。本章内容安排如下：

* [从Hello Word出发](01.md)
* [字段与方法描述符](02.md)
* [读写局部变量表与操作数栈](03.md)
* [基于对象的操作](04.md)
* [访问静态字段与静态方法](05.md)
* [调用方法的四条指令](06.md)
* [不同类型返回值对应的指令](07.md)
* [创建数组与访问数组元素](08.md)
* [条件分支语句的实现](09.md)
* [循环语句的实现](10.md)
* [异常处理的实现](11.md)
* [本章小结](12.md)

---

[^1]: 《Java虚拟机规范》Java SE 8版字 节码指令集简介

 
