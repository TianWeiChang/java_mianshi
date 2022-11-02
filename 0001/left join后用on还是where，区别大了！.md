left join后用on还是where，区别大了！

前天写`SQL`时本想通过 `A left B join on and `后面的条件来使查出的两条记录变成一条，奈何发现还是有两条。

后来发现 join on and 不会过滤结果记录条数，只会根据and后的条件是否显示 B表的记录，A表的记录一定会显示。

不管and 后面的是`A.id=1`还是`B.id=1`,都显示出A表中所有的记录，并关联显示B中对应A表中id为1的记录或者B表中id为1的记录。

运行`SQL`: 

> select * from student s left join class c on s.classId=c.id order by s.id

![img](https://img-blog.csdn.net/20180402110252327?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dxYzE5OTIwOTA2/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70) 



运行`SQL`: 

> select * from student s left join class c on s.classId=c.id and s.name="张三" order by s.id 

![img](https://img-blog.csdn.net/20180402110050921?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dxYzE5OTIwOTA2/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70) 



运行`SQL`: 

> select * from student s left join class c on s.classId=c.id and c.name="三年级三班" order by s.id 

![img](https://img-blog.csdn.net/20180402110123796?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dxYzE5OTIwOTA2/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70) 

数据库在通过连接两张或多张表来返回记录时，都会生成一张中间的临时表，然后再将这张临时表返回给用户。 

在使用`left jion`时，on和where条件的区别如下： 

1、 on条件是在生成临时表时使用的条件，它不管on中的条件是否为真，都会返回左边表中的记录。

2、where条件是在临时表生成好后，再对临时表进行过滤的条件。这时已经没有`left join`的含义（必须返回左边表的记录）了，条件不为真的就全部过滤掉。

假设有两张表： 

![1634432283467](E:\other\网络\assets\1634432283467.png)

两条`SQL`: 

> 1、select * form tab1 left join tab2 on (tab1.size = tab2.size) where tab2.name=’AAA’
> 2、select * form tab1 left join tab2 on (tab1.size = tab2.size and tab2.name=’AAA’)

![1634432327079](E:\other\网络\assets\1634432327079.png)

![1634432337077](E:\other\网络\assets\1634432337077.png)

其实以上结果的关键原因就是`left join`,`right join`,`full join`的特殊性，不管on上的条件是否为真都会返回left或right表中的记录，full则具有left和right的特性的并集。

而`inner join`没这个特殊性，则条件放在on中和where中，返回的结果集是相同的。