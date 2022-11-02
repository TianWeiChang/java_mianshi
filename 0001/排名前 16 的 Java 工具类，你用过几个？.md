排名前 16 的 Java 工具类，你用过几个？

在Java中，实用程序类是定义一组执行通用功能的方法的类。

这篇文章展示了最常用的Java实用工具类及其最常用的方法。类列表及其方法列表均按受欢迎程度排序。数据基于从GitHub随机选择的50,000个开源Java项目。

希望您可以通过浏览列表来了解流行的功能。这些方法的名称通常指示它们的作用。如果方法名不够直观，您还可以查看其他开发人员是如何在其开源项目中使用它们的。

## 1、`IOUtils`类

`org.apache.commons.io.IOUtils`类的所有成员函数都被用来处理输入 - 输出流，它的确非常利于来编写处理此类事务的程序。`IOUtils`与其他Apache Commons中的类一样，都是处理IO操作的非常重要包装器，与手动编写这些功能的其他程序相比，它们实现这些方法的代码变得更小，更清晰，更易理解。 

常见方法如下：

```
closeQuietly()  
toString()  
copy()  
toByteArray()  
write()  
toInputStream()  
readLines()  
copyLarge()  
lineIterator()  
readFully()  
```

2、`FileUtils`

`org.apache.commons.io.FileUtils`

```
deleteDirectory()  
readFileToString()  
deleteQuietly()  
copyFile()  
writeStringToFile()  
forceMkdir()  
write()  
listFiles()  
copyDirectory()  
forceDelete()  
```

3、`org.apache.commons.lang.StringUtils`

```
isBlank()  
isNotBlank()  
isEmpty()  
isNotEmpty()  
equals()  
join()  
split()  
EMPTY  
trimToNull()  
replace()  
```

4、`org.apache.http.util.EntityUtils`

```
toString()  
consume()  
toByteArray()  
consumeQuietly()  
getContentCharSet()  
```

5、`org.apache.commons.lang3.StringUtils`

```
isBlank()  
isNotBlank()  
isEmpty()  
isNotEmpty()  
join()  
equals()  
split()  
EMPTY  
replace()  
capitalize()  
```

6、`org.apache.commons.io.FilenameUtils`

```
getExtension()  
getBaseName()  
getName()  
concat()  
removeExtension()  
normalize()  
wildcardMatch()  
separatorsToUnix()  
getFullPath()  
isExtension()  
```

7、`org.springframework.util.StringUtils`

```
hasText()  
hasLength()  
isEmpty()  
commaDelimitedListToStringArray()  
collectionToDelimitedString()  
replace()  
delimitedListToStringArray()  
uncapitalize()  
collectionToCommaDelimitedString()  
tokenizeToStringArray()  
```

8、`org.apache.commons.lang.ArrayUtils`

```
contains()  
addAll()  
clone()  
isEmpty()  
add()  
EMPTY_BYTE_ARRAY  
subarray()  
indexOf()  
isEquals()  
toObject()  
```

9、`org.apache.commons.lang.StringEscapeUtils`

```
escapeHtml()  
unescapeHtml()  
escapeXml()  
escapeSql()  
unescapeJava()  
escapeJava()  
escapeJavaScript()  
unescapeXml()  
unescapeJavaScript()  
```

10、`org.apache.http.client.utils.URLEncodedUtils`

```
format()  
parse()  
```

11、`org.apache.commons.codec.digest.DigestUtils`

```
md5Hex()  
shaHex()  
sha256Hex()  
sha1Hex()  
sha()  
md5()  
sha512Hex()  
sha1()  
```

12、`org.apache.commons.collections.CollectionUtils`

```
isEmpty()  
isNotEmpty()  
select()  
transform()  
filter()  
find()  
collect()  
forAllDo()  
addAll()  
isEqualCollection()  
```

13、`org.apache.commons.lang3.ArrayUtils`

```
contains()  
isEmpty()  
isNotEmpty()  
add()  
clone()  
addAll()  
subarray()  
indexOf()  
EMPTY_OBJECT_ARRAY  
EMPTY_STRING_ARRAY  
```

14、`org.apache.commons.beanutils.PropertyUtils`

```
getProperty()  
setProperty()  
getPropertyDescriptors()  
isReadable()  
copyProperties()  
getPropertyDescriptor()  
getSimpleProperty()  
isWriteable()  
setSimpleProperty()  
getPropertyType()  
```

15、`org.apache.commons.lang3.StringEscapeUtils`

```
unescapeHtml4()  
escapeHtml4()  
escapeXml()  
unescapeXml()  
escapeJava()  
escapeEcmaScript()  
unescapeJava()  
escapeJson()  
escapeXml10()  
```

16、`org.apache.commons.beanutils.BeanUtils`

```
copyProperties()  
getProperty()  
setProperty()  
describe()  
populate()  
copyProperty()  
cloneBean()  
```

来源：www.programcreek.com/2015/12/top-10-java-utility-classes/  