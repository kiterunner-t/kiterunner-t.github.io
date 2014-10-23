---
layout: post
permalink: /blog/informix-load-chaos.html
title: "Informix load数据导入缓冲区溢出及中文乱码"
category: articles
tags: [Jekyll]
---


    kiterunner_t
    TO THE HAPPY FEW


前段时间看见同事辛辛苦苦的用load工具将数据导入Informix数据库，较大数据量导入时，经常报缓存空间不足以及因为load处理中文时导致的乱码而辛苦万分。

load是基于ASCII的Informix工具，当使用竖线 “\|” (ASCII为0x7C)作为字段分隔符时，某些汉字，如“珅”，其对应的GB18030编码为 0xAB 0x7C, 则load等UNIX工具处理时就会将该汉字第二个编码作为分隔符，而非一个整体。所以应当使用转义字符对其进行转义，即将“珅”的编码改为，0Xab 0x5C 0x7C, 这样这些UNIX工具就可以正常处理中文编码问题。（sed/awk等工具在处理时，没有对其进行转义，是这些工具没有保持一致的语义？编码真是个麻烦的问题，是多样性与复杂性之间的取舍？）

当使用 ‘!’(ASCII码为0×21) 作为字段分隔符时，不存在这种问题，因为GB18030编码的第二个以后的字节不可能出现该值。但是可能某些字段中本身就含有该字符（这种情况相对于 ‘|’ 作为分隔符引起的问题少之又少，在我们的系统里），所以在filter_5c7c进行处理时，若遇见字段中有该字符，也对其进行了转义。

需要说明的是，我们的系统Informix数据库使用系统默认的编码，即en_us.8859-1, GLS会负责进行字符转换，使中文汉字可以正常显示。因此我不知道是否当改变数据库编码时，load工具是否可以正常处理这些中文汉字。

另外，load装数时，大批量的数据常报错。可以使用dbload工具代替。这里我写了一个能正常工作的脚本和字符转义的C代码（也就仅仅能工作罢了，还有很多问题需要处理）。可能的选择是先用filter_5c7c对数据进行转换，再用dbload工具装数，但是经常数百万数千万的数据也就那么几百行有问题，这里我仅仅是做着玩，就只从dbload的错误文件中过滤出因字段不匹配（最大可能是中文乱码引起）的数据用filter_5c7c进行处理，无形中增加了脚本的复杂性。

代码参考这里 [https://github.com/kiterunner-t/krt/tree/master/t/ifx_load][1]。


[1]: https://github.com/kiterunner-t/krt/tree/master/t/ifx_load
