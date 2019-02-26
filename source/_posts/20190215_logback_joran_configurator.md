---
layout: post
title: "logback中配置解析组件joran源码解析"
date: 2019-2-15 16:59
tags:
	-  源码
	- logback
---

为了理解logback日志工具使用及源码，阅读理解官网文档、源码中的example工程、源码中的单元测试功能代码及源码、sax和jaxp的相关资料，使用planUML画类图和时序图辅助理解，
2018年前前后后看了有大半年。接下来我会根据学习进度，把自己的理解和思考记录起来。

joran使用sax及jaxp解析xml配置文件，根据我的理解，把joran源码中分为两部分：
+ 使用sax解析xml配置文件把每一个标签封装生成saxEvent对象
+ 解析上一步生成的saxEvent对象列表，执行匹配的action

<!-- more -->

## **基于SAX的XML文件解析**
![整体的时序图](/assets/blogImg/20190215/joran_total.png)
**1.时序图**
![时序图](/assets/blogImg/20190215/sax_p.png)
**2.类图**
![类图](/assets/blogImg/20190215/sax.png)

**3.XML解析过程**
 1. GenericConfigurator初始化，设置Context和rulesMap；（用户端）
 2. 传入配置配置文件，以文件url为systemId、文件的inputStream生成SAX解析的inputSource对象；（框架处理）
 3. 使用context生成SaxEventRecorder对象recorder，SaxEventRecorder继承DefaultHandler类具有SAX解析xml文件的能力；（框架处理）
 4. 调用recorder的recordEvents方法传入inputSource对象，解析xml文件 [SAX解析实例](https://blog.csdn.net/andie_guo/article/details/24851077)；（框架处理）
 5. 调用SAX接口按顺序解析xml文件中的标签，按顺序逐个封装为SaxEvent对象保存在list列表中；（框架处理）

SaxEvent的子类：
+ StartEvent ：封装标签的namespaceURI, localName, qName, attributes, locator, elementPath（封装标签路径对象）属性。前五个属性皆为SAX解析产生，elementPath为joran封装的当前标签所在位置。
+ BodyEvent：封装标签体的text，locator属性。
+ EndEvent ：封装结束标签的namespaceURI, localName, qName, locator属性。

 6. 解析完成后，效果为把磁盘上的xml配置文件转换为*内存中有对应顺序可直接读取操作的SaxEvent对象集合*；（框架处理）
 
 		
**4.传入参数和返回值**
inc.xml配置文件

```
<?xml version="1.0" encoding="UTF-8" ?>

<x xmlns="http://www.testsax.org/schema/beans"
	xmlns:p="http://www.testsax.org/schema/p">
  <p:inc increment="1"/>
  <p:first isleaf="false" value="value">
  	<second isleaf="true">
  		second content text
  	</second>
  </p:first>
</x>
```

joran解析后封装的SaxEvent集合

```
[
	StartEvent(qName:"x";			localName:"x";		namespaceURI:"http://www.testsax.org/schema/beans") ([x]) [4,44], 	
	StartEvent(qName:"p:inc";		localName:"inc";		namespaceURI:"http://www.testsax.org/schema/p";attributes:{increment="1"}) ([x][inc]) [5,25],   
	EndEvent  (qName:"p:inc";		localName:"inc;		namespaceURI:"http://www.testsax.org/schema/p"")  [5,25], 
	StartEvent(qName:"p:first";	localName:"first";	namespaceURI:"http://www.testsax.org/schema/p";attributes:{isleaf="false" value="value"}) ([x][first]) [6,41], 
	StartEvent(qName:"second";		localName:"second";	namespaceURI:"http://www.testsax.org/schema/beans";attributes:{isleaf="true"}) ([x][first][second]) [7,26], 
	BodyEvent (second content text)  [9,4],   
	EndEvent  (qName:"second";		localName:"second";	namespaceURI:"http://www.testsax.org/schema/beans")  [9,13],   
	EndEvent  (qName:"p:first";	localName:"first";	namespaceURI:"http://www.testsax.org/schema/p")  [10,13],   
	EndEvent  (qName:"x";			localName:"x";		namespaceURI:"http://www.testsax.org/schema/beans")  [11,5]
]
```


**5.注意**
1.xml中完整的标签包含开始标签、文本、结束标签，在joran中封装为StartEvent、BodyEvent、EndEvent对象；
2.sax是按照顺序解析xml中的标签，解析完成生成的SaxEvent列表的顺序反映了标签的顺序；
3.每一个StartEvent中ElementPath属性对象封装该标签在xml文件中的路径，如：*[x][inc]*；
4.*在解析xml标签时使用了栈数据结构。*在实现中有一个全局的globalElementPath对象作为栈，在解析到开始标签时压入该标签名，
	在解析到结束标签时弹出最上面的标签名，这样实现了在解析到每一个开始标签时，当前栈中的标签顺序就是该开始标签在xml文件中的真实顺序；
5.joran使用ArrayList实现了栈的push(String), pop(), peekLast()操作；

**6设计**
1.对于StartEvent、EndEvent、BodyEvent都需要设置标签对应Locator对象的副本属性，所以在类图设计上在父类SaxEvent的构造函数设置，
子类调用父类的构造函数初始化；
2.对于一个完整的标签，只有开始标签中有Attributes属性，另外只用在开始标签设置标签路径elementPath属性，在程序中就可以找到完整的标签，
所以在开始标签StartEvent类中添加attributes、elementPath两个独有的属性。在生成StartEvent对象时完全拷贝attributes对象；
3.对SaxEvent类及其子类的设计需要考虑到XML协议及SAX解析机制（查看小实例理解）；

**7.问题及思考**
+ 如何在SaxEvent列表中找到一个开始标签对应的完整标签内容，如p:first标签。

+ 查找栈的实现方式，通过joran对栈的使用理解这种数据结构应用及存在的意义，进而思考其他数据结构如队列、链表。

+ 一个清晰的toString可以帮助理解输出。

+ 在封装SaxEvent对象时使用了两处深拷贝，一个是对Locator属性，一个是ElementPath属性。

## **匹配执行Action**

解析SaxEvent列表的目的是为了查找匹配的标签执行对应的rule规则

**1.类图及时序图**
![时序图](/assets/blogImg/20190215/saxevent_p.png)
![类图](/assets/blogImg/20190215/saxevent.png)

**2.SaxEvent列表解析过程**
1. 初始化RuleStore、Interpreter、InterpretationContext、EventPlayer、Action等相关对象及其依赖；
2. 遍历SaxEventList列表，并根据每一个对象类型，调用interpreter解析器中不同的方法进行处理；
3. interpreter解析器的elementPath（标记解析当前SaxEvent完整路径）、locator（标记解析当前SaxEvent的行列）、actionListStack栈（存放标签匹配的Action列表）三个属性；
4. interpreter解析器对StartEvent对象（代表xml标签的开始标签）的解析，在属性elementPath栈中放入当前标签（栈中的所有值为当前标签的完整路径），
	根据当前标签的完整路径查找匹配的Action列表，遍历Action列表并回调Action的begin(InterpretationContext ic, String tagName, Attribute atts)函数；
5. interpreter解析器对BodyEvent对象（代表xml标签文本内容text）的解析，在actionListStack栈中peek()出当前标签的匹配Action列表元素，
	遍历Action列表并回调Action的body(InterpretationContext ic, String body)方法；
6. interpreter解析器对EndEvent对象（代表xml标签的结束标签）的解析，从actionListStack栈中弹出pop当前标签的匹配Action列表元素，
	遍历Action列表并回调Action的end(InterpretationContext ic, String name)方法，elementPath栈弹出pop栈顶元素；

**3.设计**
1. 对interpreter和EventPlayer两个类的关系及设计意图的理解
	interpreter中有对SaxEvent每一种类型的处理方法，如：startElement(StartEvent)、characters(BodyEvent)、endElement(EndEvent)在这些方法中处理对应的SaxEvent并执行匹配的Action，
	interpreter中有各种SaxEvent类型处理需要公用的一些属性，如：elementPath、locator、actionListStack、skip（路径解析出错，跳过该ElementPath及其子路径使用）用于共享当前标签的一些属性。
	EventPlayer类可以看做为SaxEvent处理的策略整合类，对于每一种类型的SaxEvent除了需要调用Interpreter中的处理方法外还需要调用InterpretationContext类中的处理方法，而且以后这个处理策略有可能修改。
2. Interpreter与InterpretationContext两个类关系*此两个用法找一个示例*
	在Interpreter类中有InterpretationContext的引用依赖，在InterpretationContext类中也有Interpreter的引用依赖，所以他两个是双向依赖；目前有两处使用InterpretationContext属性如下：
	在Interpreter解析器中解析执行ImplicitAction是会传入interpretationContext引用，在EventPlayer中的play()方法中对于saxEventlist中的每一个元素都调用了InterpretationContext.fireInPlay(se);
3. Interpreter类中ElementPath类型的skip属性如何跳过某一个标签及子标签说明
```
<a>
	<b skip="true">
		<c>text</c>
	</b>
</a>
```
	
有如上的xml标签结构，由于某种原因（可能是自己设置，或者是该节点Action执行出错需要跳过）需要跳过b标签及它的子标签解析，使用elementPath栈和skip栈实现的解析过程如下：

1.初始状态elementPath为空栈，skip为null，开始解析 遇到开始标签\<a\>，elementPath.push(a)，判断skip是否为null，是，则执行接下来的查找匹配Action并执行回调操作；
2.遇到开始标签\<b\>, elementPath.push(b), skip = elementPath.duplicate()，判断skip是否为null，否，则跳过查找Action及接下来的执行回调操作;
3.遇到开始标签\<c\>, elementPath.push(c), 判断skip是否为null，否，则跳过查找Action及接下来的执行回调操作;
4.遇到c标签文本text，判断skip是否为null，否，则跳过查找Action及接下来的执行回调操作;
5.遇到结束标签\</c\>, 判断skip是否为null, 否，则跳过查找Action及接下来的执行回调操作，并判断elementPath.equals(skip)，否，skip保持不变，elementPath.pop()；
6.遇到结束标签\</b\>, 判断skip是否为null, 否，则跳过查找Action及接下来的执行回调操作，并判断elementPath.equals(skip)，是，skip设置为null，elementPath.pop()；
7.遇到结束标签\</a\>, 判断skip是否为null， 是，则执行接下来的查找匹配Action并执行回调操作，elementPath.pop()；

如上步骤则跳过了标签b的开始结束标签，c的开始结束标签及标签文本text节点执行匹配的Action的回调操作。

4. Action的解耦及Action中接口方法抛出异常在解析时处理的设计

logback使用，用户只用提供一个logback.xml的配置文件，然后在程序需要记录日志的地方直接调用logback有限的几个方法就可以记录日志，对于日志格式化、保存位置、log级别设置等很多特性都需要logback框架从配置文件获取设置信息并处理，然后根据用户的设置保存日志，这样使用者和logback框架实现了最小的耦合，日志的实际处理都内聚到框架内部（关键是要洞悉用户日志记录的主要真实需求）。*获取那些信息、进行何种处理、这些处理之间关联关系*  对于这些joran是如何处理它们的耦合和内聚关系的？

joran在SaxEventRecorder中通过SAX解析xml文件，把xml文件中的标签、属性、命名、标签路径、行列位置信息、标签内容封装在保存SaxEvent对象的SaxEventList列表中。列表中元素的顺序体现标签的路径，对于如何在遍历SaxEventList列表时获取到标签路径信息的实现过程则封装在Interpreter解析器的实现中。这样用*SaxEventList*就把*xml文件解析到内存中*和*获取内存中标签信息并处理*这两个过程解耦，然后具体的处理过程内聚到这两个过程中。

对于logback提供的日志格式化等功能都是体现在配置文件中的不同标签、标签属性、标签内容中，且各个功能中的关联性不强。所以只要能*遍历配置中的各个标签，识别各个功能的配置设置，然后执行对应功能的处理程序*。之前已经把xml配置文件解析到内存中的SaxEventList中，现在只需要遍历这个列表即可获取和识别标签信息。

logback的一个功能在配置文件中的体现通常是一个标签路径下的多个标签及标签内容属性，在joran中它把这些功能的处理封装到Action接口类中，Action中的接口如下：

```java
public abstract class Action extends ContextAwareBase {
	public abstract void begin(InterpretationContext ic, String name, Attributes attributes) throws ActionException;
	public void body(InterpretationContext ic, String body) throws ActionException {
    	// NOP
	}
	public abstract void end(InterpretationContext ic, String name) throws ActionException;
	
}
```

然后使用标签的路径ElementSelector与其对应的所有Action（一系列的ElementSelector组合起来代表一个功能），来表示xml配置文件中这个功能所要进行的处理（这样把一个功能如何通过Action的几个接口解析处理的可在以后具体功能源码解析中分析）。这样用*ElementSelector和Action*这两个类把*从配置信息中解析各个功能的配置*与*配置的功能的处理过程（封装到Action中）*这两个过程解耦，功能的处理过程内聚到Action实现类中。
	在上面两个逻辑解耦都牵扯到封装xml元素的SaxEvent及其子类StartEvent、BodyEvent、EndEvent的设计
	+ 在SaxEventRecorder中是用SAX把标签相关信息封装为SaxEvent的实现类保存在SaxEventList列表中。
	+ 在Interpreter解析器中遍历SaxEventList列表每一个元素并使用栈解析出当前元素的路径，使用到判断StartEvent类型时入栈，EndEvent类型时出栈。
	+ 在Action对功能的处理程序中，判断SaxEvent的类型来执行begin、body、end接口。
	在Action中处理方法抛出异常时，在Interpreter解析器中捕获这个异常并根据异常类型使用框架自身的日志记录。
	
5. Interpreter与InterpretationContext设计定位

根据InterpretationContext在Action的使用开来，它就像是Interpreter的上下文环境，有它可以在Action中获得Interpreter的引用；根据InterpretationContext在EventPlayer.play(aSexEventList)方法中的使用，看它可以在调用Action中方法时先执行一些操作或后执行一些操作，类似包装设计模式的感觉，对于它更深入的理解有待于接下来对具体功能的分析。

Interpreter为解析器，joran使用SAX通过SaxEventList把*xml配置文件解析到内存*和*遍历配置执行相关操作*解耦，使用Action和ElementSelector把*解析内存中功能对应的配置*与*根据相应配置执行功能的相关操作*解耦。对于内存中封装配置信息的SaxEventList列表的遍历、解析路径、根据路径匹配规则处理的操作就在Interpreter解析器中进行的，要进行这些操作它需要关联保存规则的RuleStore的引用和InterpretationContext的引用。
	
	
**4.注意**
+ 由于joran使用SAX解析时xml文件时是顺序解析的，所以SaxEventList列表中项的顺序可以反映xml文件中标签的顺序，
	对于一个标签的开始标签（StartEvent）、标签内容text（BodyEvent）、结束标签（EndEvent）对应的完整路径应该是一样的，在Interpreter中用elementPath栈来标记当前标签的完整路径，实现如下：
	在按顺序遍历SaxEventList时，判断当前元素类型为StartEvent时，将当前标签名称推入elementPath栈中，遇到EndEvent类型元素时，在elementPath栈栈中弹出元素。
	这样可以保证在当前标签的开始标签、标签的文本BodyEvent、结束标签和该标签的子标签中，通过获取Interpreter类elementPath栈中的所有元素作为当前元素的完整路径名。
	
+ 对于一个标签的开始标签、标签内容text、结束标签都需要应用匹配该标签路径的Action，和确定标签的完整路径一样使用栈，在Interpreter中使用Stack<List<Action>> actionListStack栈，实现如下：
	在按顺序遍历SaxEventList时，判断当前元素为StartEvent时，可以通过elementPath栈获取完整路径，然后通过一定规则可以获取一个Action列表，把它压入actionListStack栈中，遇到EndEvent类型元素时，
	在actionListStack中弹出栈顶元素。这样在该标签的开始标签、标签文本内容、结束标签都可以通过actionListStack.peek()获取匹配的Action列表，执行相应规则。

**5.问题及思考**
+ 最近开发工作表生成图表功能的相关方法时，发现方法的参数很多很长，如何减少参数？
	1. 在看到joran解析SaxEvent源码buildInterpreter()方法中调用addImplicitRules(interpreter)只传入一个参数，使我对继承封装有点新理解。
	TrivialConfigurator类继承GenericConfigurator类并实现了addImplicitRules(Interpreter)虚方法,
	在该方法中它可以访问该类的属性，如：rulesMap，另外由于Interpreter关联到RuleStore和InterpretationContext，所以通过传参可以访问这三个类中的方法和属性。
	这样就不需要方法体中需要某个值就给方法添加一个参数，可以从当前方法所在类的属性、方法参数对象、通过方法参数关联的对象中的方法和属性获取到需要的值。
	这就需要对需求有深刻的理解，把需求抽象为对象并设计这些对象之间的关联关系。
	2. 在Interpreter类的startElement()方法处理中，获取标签所有符合的Action的方法getApplicableActionList只传入了elementPath和atts两个参数，
	在getApplicableActionList(elementPath, atts)方法中可以通过该方法所在的Interpreter类获取类属性ruleStore和interpretationContext两个属性，所以通过类缩减了方法参数，
	另外通过这两个属性所属的类中的方法来具体执行查找Action的操作，缩减了该方法体的长度。