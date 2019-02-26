---
date: 2019-02-25 16:43
layout: post
title: joran的interpreter解析器解析SaxEvent及匹配Action
tags:
	- logback
	-  源码
---

interpreter解析器解析StartEvent、EndEvent、BodyEvent及查找匹配指定路径的标签对应的Action部分源码理解及梳理

标签路径相关接口和匹配策略在RuleStore接口及其子类中定义，*第二种匹配策略应该存在bug*

<!-- more -->

## **路径匹配规则**
当前路径elementPath ep：  a/b/c
规则对应的路径elementSelector es：（优先级按照如下顺序查找匹配的Action，找到则返回）

**1. 按照当前路径和规则路径完全匹配**
``fullPathMatch(ElementPath elementPath)``
+ a/b/c				&nbsp;&nbsp;	<
+ d/a/b/c
+ a/b

**2. 第一个是*通配符, 然后从后向前查找匹配路径最多的(可能有bug，测试如下情况选择了第三种，其实一种才是正确的匹配情况)**
``suffixMatch(ElementPath elementPath)``
+ \*/b/c			&nbsp;&nbsp;	2	<?
+ \*/d/a/b			&nbsp;&nbsp;	0
+ \*/d/e/b/c		&nbsp;&nbsp;	2	<?	<

```
List<Action> suffixMatch(ElementPath elementPath) {
    int max = 0;
    ElementSelector longestMatchingElementSelector = null;

    for (ElementSelector selector : rules.keySet()) {
        if (isSuffixPattern(selector)) {
            int r = selector.getTailMatchLength(elementPath);
   		// bug修复：此处改为应添加(selector.size() - 1 == r)判断
   		// if (selector.size() - 1 == r && r > max) {
            if (r > max) {
                max = r;
                longestMatchingElementSelector = selector;
            }
        }
    }

    if (longestMatchingElementSelector != null) {
        return rules.get(longestMatchingElementSelector);
    } else {
        return null;
    }
}
```
**3. 最后一个是*通配符, 然后从前查找匹配路径最长并且es.size() - 1等于最大的匹配长度 **
``prefixMatch(ElementPath elementPath)``

+ d/a/b/c/*			&nbsp;&nbsp;	0
+ d/a/b/c/e/*		&nbsp;&nbsp;	0
+ b/c/e/*			&nbsp;&nbsp;	0
+ a/b/*				&nbsp;&nbsp;	2
+ a/b/c/*			&nbsp;&nbsp;	3	&nbsp;&nbsp;	<
+ a/b/c/d/*			&nbsp;&nbsp;	3

```
// 注意：r == selector.size() - 1)
// 加上这个判断是为了完全匹配
List<Action> prefixMatch(ElementPath elementPath) {
    int max = 0;
    ElementSelector longestMatchingElementSelector = null;

    for (ElementSelector selector : rules.keySet()) {
        String last = selector.peekLast();
        if (isKleeneStar(last)) {
            int r = selector.getPrefixMatchLength(elementPath);
            // to qualify the match length must equal p's size omitting the '*'
            if ((r == selector.size() - 1) && (r > max)) {
                max = r;
                longestMatchingElementSelector = selector;
            }
        }
    }

    if (longestMatchingElementSelector != null) {
        return rules.get(longestMatchingElementSelector);
    } else {
        return null;
    }
}
```
**4. 第一个和最后一个都是通配符, 然后匹配规则路径去除前后的*通配符, 判断当前路径ep.contain(es), 如果包含去es.size()最大的匹配路径的规则返回**
``middleMatch(ElementPath path)``（可能存在bug, 通过ElementSelectorDemo测试，无bug，注意如下代码）

+ \*/a/b/c/*			&nbsp;&nbsp;	3	<	&nbsp;&nbsp;	3
+ \*/d/a/b/c/*			&nbsp;&nbsp;	4	<?	&nbsp;&nbsp;	0
+ \*/b/c/*				&nbsp;&nbsp;	0		&nbsp;&nbsp;	2
+ \*/b/*				&nbsp;&nbsp;	0		&nbsp;&nbsp;	1
+ \*/a/b/c/e/*			&nbsp;&nbsp;	4	<?	&nbsp;&nbsp;	0
+ \*/*					&nbsp;&nbsp;	0		&nbsp;&nbsp;	0
+ \*/d/a/b/c/e/*		&nbsp;&nbsp;	5	<?	&nbsp;&nbsp;	0
+ \*/d/e/a/b/c/*		&nbsp;&nbsp;	5	<?	&nbsp;&nbsp;	0
+ \*/a/b/c/d/e/*		&nbsp;&nbsp;	5	<?	&nbsp;&nbsp;	0

```
// SimpleRuleStore类
List<Action> middleMatch(ElementPath path) {

    int max = 0;
    ElementSelector longestMatchingElementSelector = null;

    for (ElementSelector selector : rules.keySet()) {
        String last = selector.peekLast();
        String first = null;
        if (selector.size() > 1) {
            first = selector.get(0);
        }
        if (isKleeneStar(last) && isKleeneStar(first)) {
            List<String> copyOfPartList = selector.getCopyOfPartList();
            if (copyOfPartList.size() > 2) {
                copyOfPartList.remove(0);
                copyOfPartList.remove(copyOfPartList.size() - 1);
            }

            int r = 0;
            ElementSelector clone = new ElementSelector(copyOfPartList);
            if (clone.isContainedIn(path)) {
                r = clone.size();
            }
            if (r > max) {
                max = r;
                longestMatchingElementSelector = selector;
            }
        }
    }

    if (longestMatchingElementSelector != null) {
        return rules.get(longestMatchingElementSelector);
    } else {
        return null;
    }
}

// ElementSelector类
// 注意：isContainedIn方法中是：当前路径p.contents(规则中的路径elementSelector)，而不是相反。说明如下：
// 例如当前路径 p="/a/b/c" 
// 规则中匹配路径 ps1="*/d/a/b/c/e/*", ps2="*/a/b/*
// ps1和ps2去除前后的*通配符后，执行p.contains(ps2)，则路径ps2的规则匹配
public boolean isContainedIn(ElementPath p) {
    if (p == null) {
        return false;
    }
    return p.toStableString().contains(toStableString());
}
```

## **设计**
**1. 对*xml标签元素路径*和*路径选择*两个需求的封装到ElementPath类和ElementSelector类的理解**

&ensp;需要考虑解决的需求问题：
在使用sax解析xml文件为内存中的``SaxEventList``时需要考虑除了要保存标签内容、属性、名称、命名空间等这些封装到sax解析的过程中了，不需要joran框架考虑，
在joran中需要处理的是，解析后封装的``SaxEventList``中体现这些标签的所在位置（也就是路径，可以找到当前标签的父标签和子标签），这样可以通过对不同的路径的xml标签元素运用不同的处理策略（也就是Action）。
上面的问题清楚后需要对每一个标签执行匹配的规则，现在遍历``SaxEventList``列表，可以获取到当前SaxEvent元素的内容（包括：标签名、属性、命名空间、行列信息、路径信息）和所有的匹配规则ruleStore，设计时需要考虑解决的问题是：
对于saxEvent元素的路径，根据怎样的规则，匹配ruleStore规则中的路径，然后执行对应的规则。
	
&ensp;joran解决这一问题的设计如下：
在SAX解析xml时是按照顺序解析的，joran使用``ArrayList saxEventList``来保存封装标签的SaxEvent元素，然后``saxEventList``中的顺序就体现了标签的路径；然后在Interpreter解析器中匹配规则的处理时，根据遍历到的SaxEvent元素的类型，使用一个栈类型的全局变量来解析当前元素的路径。
由于所有匹配规则保存在RuleStore中，所以在这个接口类中提供了``matchActions(ElementPath elementPath)``方法传入路径获取当前路径匹配的规则列表``List<Action>``，把具体的路径匹配处理过程封装起来在实现类SimpleRuleStore中实现。

&ensp;路径匹配需求（是否有*通配符及通配符所在位置）确定四种匹配规则，按照优先级（按优先级只要有一个匹配上就返回）如下：
+ 无*通配符，完全匹配；
+ 前面有*通配符， 从后向前 查找去除通配符后匹配深度最长的，且路径长度等于匹配深度``(elementSelector.size() - 1 == r)``；
	&ensp;如当前路径为/a/b/c, 匹配路径如下：
	1. \*/b/c			&nbsp;&nbsp;	2	&nbsp;&nbsp;	<?	&nbsp;&nbsp;	<
	2. \*/d/a/b			&nbsp;&nbsp;	0
	3. \*/d/e/b/c		&nbsp;&nbsp;	2	&nbsp;&nbsp;	<?	&nbsp;&nbsp;	``(elementSelector.size() - 1 = 3)``
+ 后面有*通配符， 从前向后 查找去除通配符后匹配深度最长的，且路径长度等于匹配深度``(elementSelector.size() - 1 == r)``；
	&ensp;如当前路径为/a/b/c, 匹配路径如下：
	1. /a/c/\*			&nbsp;&nbsp;	1
	2. /b/a/b/c/\*		&nbsp;&nbsp;	0
	3. /a/b/\*			&nbsp;&nbsp;	2	<
	4. /a/b/e/\*		&nbsp;&nbsp;	2		&nbsp;&nbsp;	``(elementSelector.size() - 1 = 3)``
+ 前后都有*通配符，去除前后的通配符，查找当前路径包含该路径则匹配的情况下，该路径最长的情况；
	&ensp;如当前路径为/a/b/c, 匹配路径如下：
	1. \*/a/b/c/\*		&nbsp;&nbsp;	3	&nbsp;&nbsp;	<
	2. \*/b/c/\*		&nbsp;&nbsp;	2
	3. \*/d/b/c/\*		&nbsp;&nbsp;	0
+ 按照以上优先级先后顺序还未找到匹配的规则，则返回null；

**2. 接口和实现类关系(RuleStore与SimpleRuleStore),  实现类与具体的处理类关系(SimpleRuleStore与ElementSelector)**

&ensp;RuleStore接口设计的目的就是保存和查找匹配规则，所以它提供了如下两个添加rule的接口：
```java
public interface RuleStore {
	void addRule(ElementSelector elementSelector, String actionClassStr) throws ClassNotFoundException;
	void addRule(ElementSelector elementSelector, Action action);
	List<Action> matchActions(ElementPath elementPath);
}
```

&ensp;在其实现类SimpleRuleStore中需要考虑：
+ 把rule保存在哪里
	保存在``HashMap<ElementSelector, List<Action>> rules``，根据指定的匹配路径把Action分组。
+ 指定路径如何与rule关联起来，即：给定一个路径如何找到它的所有规则
	路径匹配的规则算法有四种，根据优先级把查找指定路径的Action封装到matchActions(ElementPath elementPath)方法中
```java
public List<Action> matchActions(ElementPath elementPath) {
    List<Action> actionList;
    if ((actionList = fullPathMatch(elementPath)) != null) {
        return actionList;
    } else if ((actionList = suffixMatch(elementPath)) != null) {
        return actionList;
    } else if ((actionList = prefixMatch(elementPath)) != null) {
        return actionList;
    } else if ((actionList = middleMatch(elementPath)) != null) {
        return actionList;
    } else {
        return null;
    }
}
```

&ensp;把相关更具体的处理过程提取封装到ElementSelector类中，封装为三个方法：
```java
class ElementSelector extends ElementPath {
	// 全匹配
	public boolean fullPathMatch(ElementPath path) {}
	// */a/b/c
	public int getTailMatchLength(ElementPath p) {}
	// */a/b/c/*
	public boolean isContainedIn(ElementPath p) {}
	// /a/b/c*
	public int getPrefixMatchLength(ElementPath p) {}
}
```

&ensp;这样的通过接口类（RuleStore）、实现类（SimpleRuleStore）、路径匹配类（ElementSelector其实就是一个工具类，但命名更符合领域编程）。对于查找路径对应的所有Action需求，

通过RuleStore中的matchActions接口暴露给外界，外界只用关心出入值和返回值不用关心具体的匹配规则，在joran内部通过实现类SimpleRuleStore来实现rule的保存和匹配的具体工作。然后ElementSelector把更细的，涉及到对ElementPath类的操作，提取到ElementSelector（ElementPath的子类）中，这样可以使用继承的好处（可以减少方法的形参数量、在方法内部可以直接调用类的方法），*模块内部高内聚，模块外部低耦合*；
```java
// ElementSelector类内方法
// 减少了一个形参this，在方法内部直接使用toStableString()，调用this的相关方法
public boolean isContainedIn(ElementPath p) {
    if (p == null) {
        return false;
    }
    return p.toStableString().contains(toStableString());
}
```

**3. 节点路径类ElementPath和ElementSelector**

&ensp;ElementPath封装了xml标签中特定标签的路径信息，它通过添加的顺序来确定当前标签的具体路径，这样就可以通过使用栈数据结构的推入（StartEvent类型推入）、弹出操作（EndEvent类型弹出），在遍历有顺序的SaxEventList列表确定*当前遍历元素*的路径（栈中当前所有的元素就是就为它的路径）。这有一个问题就是需要确定一个栈推入和弹出的触发条件，找到这个条件后就可以把*数据*和*处理程序*分开，在任何时候只要给了SaxEventList列表就可以传入程序进行解析。

&ensp;这样ElementPath封装元素SaxEvent的路径信息，SAX解析xml把标签封装为SaxEvent对象并存入保存顺序的SaxEventList列表中，SaxEventList的顺序可以解析出来路径，解析过程在Interpreter解析中。这样就把所有的过程解耦开了，处理逻辑内聚到Interpreter解析器、ElementPath路径、``SaxEventRecorder.recordEvents(inputSource)``使用SAX解析xml并生成SaxEventList三个部分，*模块内部高内聚，模块外部低耦合*。

&ensp;ElementSelector封装了对ElementPath路径的匹配选择，需要操作ElementPath中保存路径节点的列表，所以它如果可以直接访问``ElementPath.partList``属性和方法会更方便。所以它集成了ElementPath，是它的子类。