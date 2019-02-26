---
layout: post
title: "joran中使用SAX解析xml部分内容"
date: 2019-2-21 13:54
tags:
	- 源码
	- logback
	- sax
	- jaxp
---

joran使用SAX解析xml配置文件，此处是SAX的使用实例，及joran中如何使用SAX的，辅助理解SaxEvent及相关类的设计。

<!-- more -->

## **实例xml文档**

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
## **java代码**
新建SaxEventRecorder类继承DefaultHandler类
*SAX使用的关键代码说明如下：*

```java
final public void recordEvents(InputStream inputStream) {
    recordEvents(new InputSource(inputStream));
}
// 启动SAX,使用SAXParser解析xml文件
public List<SaxEvent> recordEvents(InputSource inputSource) {
    SAXParser saxParser = buildSaxParser();
    try {
        saxParser.parse(inputSource, this);
        return saxEventList;
    } catch (IOException ie) {
        log.info("I/O error occurred while parsing xml file", ie);
    } catch (SAXException se) {
        throw new RuntimeException("Problem parsing XML document. See previously reported errors.", se);
    } catch (Exception ex) {
    	throw new RuntimeException("Unexpected exception while parsing XML document.", ex);
    }
    throw new IllegalStateException("This point can never be reached");
}
    
// 初始化SAXParserFactory返回SAXParser
private SAXParser buildSaxParser() {
    try {
        SAXParserFactory spf = SAXParserFactory.newInstance();
        spf.setValidating(false);
        spf.setNamespaceAware(true);
        return spf.newSAXParser();
    } catch (Exception pce) {
        String errMsg = "Parser configuration error occurred";
        throw new RuntimeException(errMsg, pce);
    }
}

// 执行入口
public static void main(String[] args) throws FileNotFoundException {
	SaxEventRecorder recorder = new SaxEventRecorder();
	recorder.recordEvents(new FileInputStream(new File("src/main/resources/chapters/onJoran/inc.xml")));
	log.info("saxeventList: {}", recorder.getSaxEventList());
	System.out.println(recorder.getSaxEventList());
}
```
*SAX是基于事件触发的，在解析的过程中根据解析的内容分别触发调用不同的函数处理*
+ 在解析开始时SAX会调用一次setDocumentLocator(Locator l)把locator暴露出来，随着解析的进行程序可以通过该对象获取lineNumber和columnNumber的值

```
public void setDocumentLocator(Locator l) {
    locator = l;
}
```

+ SAX在解析到xml文档根标签时,调用DefaultHandler类的startDocument()方法

+ SAX在解析到xml的开始标签标签时,调用DefaultHandler类的startElement(String namespaceURI, String localName, String qName, Attributes atts)方法

```
public void startElement(String namespaceURI, String localName, String qName, Attributes atts) {
    String tagName = getTagName(localName, qName);
    globalElementPath.push(tagName);
    ElementPath current = globalElementPath.duplicate();
    saxEventList.add(new StartEvent(current, namespaceURI, localName, qName, atts, getLocator()));
}
```

+ SAX在解析到xml标签文本text时,调用DefaultHandler类的characters(char[] ch, int start, int length)方法

```
public void characters(char[] ch, int start, int length) {
    String bodyStr = new String(ch, start, length);
    SaxEvent lastEvent = getLastEvent();
    if (lastEvent instanceof BodyEvent) {
        BodyEvent be = (BodyEvent) lastEvent;
        be.append(bodyStr);
    } else {
        // ignore space only text if the previous event is not a BodyEvent
        if (!isSpaceOnly(bodyStr)) {
            saxEventList.add(new BodyEvent(bodyStr, getLocator()));
        }
    }
}
```
+ SAX在解析到xml的结束标签时,调用DefaultHandler类的endElement(String namespaceURI, String localName, String qName)方法

```
public void endElement(String namespaceURI, String localName, String qName) {
    saxEventList.add(new EndEvent(namespaceURI, localName, qName, getLocator()));
    globalElementPath.pop();
}
```

## **解析封装结果**

```
[
	StartEvent(qName:"x";localName:"x";namespaceURI:"http://www.testsax.org/schema/beans") ([x]) [4,44], 
	StartEvent(qName:"p:inc";localName:"inc";namespaceURI:"http://www.testsax.org/schema/p";attributes:{ increment="1"}) ([x][inc]) [5,25],   
	EndEvent  (qName:"p:inc";namespaceURI:"http://www.testsax.org/schema/p";localName:"inc")  [5,25], 
	StartEvent(qName:"p:first";localName:"first";namespaceURI:"http://www.testsax.org/schema/p";attributes:{ isleaf="false" value="value"}) ([x][first]) [6,41], 
	StartEvent(qName:"second";localName:"second";namespaceURI:"http://www.testsax.org/schema/beans";attributes:{ isleaf="true"}) ([x][first][second]) [7,26], 
	BodyEvent (second content text)  [9,4],   EndEvent  (qName:"second";namespaceURI:"http://www.testsax.org/schema/beans";localName:"second")  [9,13],   
	EndEvent  (qName:"p:first";namespaceURI:"http://www.testsax.org/schema/p";localName:"first")  [10,13],   
	EndEvent  (qName:"x";namespaceURI:"http://www.testsax.org/schema/beans";localName:"x")  [11,5]
]
```