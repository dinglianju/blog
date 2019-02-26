---
date: 2019-02-26 16:43
layout: post
title: ogback框架内日志处理(基于v_1.3.0-alpha4版本源码)
tags:
	- logback
	-  源码
---

logback日志框架对外部使用者提供日志记录功能的封装，在框架内部处理出现的异常和记录日志信息，它使用StatusManager来处理。

<!-- more -->


## **1.对Logback中status的理解**
logback是一个日志处理框架，只要在项目中引入logback的jar包依赖、正确填写日志的配置文件，然后就可以在项目引入log进行日志记录。这样在项目中就可以吧日志信息的格式化，日志信息保存位置，日志输出级别的复杂性交给日志框架来处理。status的作用也是记录日志信息，只不过它是框架内部对于项目的日志配置文件解析和校验以及项目对日志接口的使用出现错误时，把错误信息输出到控制台通知调试人员。
	
## **2.类图**
![status_print类图](/assets/blogImg/20190226/status_print_class_diagram.png)

## **3.时序图**
![status_print时序图](/assets/blogImg/20190226/status_print_sequence_chart.png)

## **4.处理逻辑**
+ 所有实现ContextAware接口的类，可以调用addWarn、addInfo、addError接口把框架在解析运行过程中出现的日志信息包装成Status（message, level, throwable, date, origin, childrenList）对象，这样实现可以记录一条日志在什么时间（date）、什么地方（origin）、什么内容（message throwable）、与其他日志的关系信息包装到Status中；
+ 在客户端中调用addWarn、addInfo、addError接口包装日志信息到Status对象中；
+ 在ContextAware的addStatus(Status status)接口中从context中获取StatusManager对象；
+ 把包装日志信息的Status添加到StatusManager对象的集合中；
+ 在客户端需要查看日志信息时调用StatusPrinter类中的printInCaseOfErrorsOrWarnings(context)接口，从context中获取StatusManager对象；
+ 从StatusManager对象中获取保存的所有日志status；
+ 根据日志级别和时间过滤status集合，并格式化每条信息调用printStream的print输出；
	
## **5.功能模块**
**重要模块**
	+ StatusManager及其子类是status-print模块的中心，负责日志status的存储管理。
	+ Status及其子类继承体系抽象了要记录的日志，以方便表示、存储、使用。
	+ contextAware及其子类，在定义了记录日志的接口，整个框架中大部分模块（Action等）多有继承，方便在有记录日志需求的地方使用。
	+ StatusPrinter和StatusUtil工具类，在需要输出已记录的日志时通过定义的静态方法方便打印日志到控制台等其他终端，判断指定时间内指定级别日志是否存在。
	+ Context的子类ContextBase中保存有StatusManager的引用，以方便在整个框架中获取。
	
**性能优化**
	在BasicStatusManager类中，当日志条数超过150条时，把接下来的日志添加到tailBuffer的CyclicBuffer环形数组中（对CyclicBuffer类的理解）
	接口实现、抽象类、子类
	+ ContextAware接口中定义了必要的方法接口，在子类ContextAwareBase中实现ContextAware接口中方法，根据接口中方法的作用意义，ContextAwareBase中引入Context的依赖；
	+ Status接口中国定义日志需要的必要方法（添加，删除，遍历等）及日志的三个级别，在子类StatusBase中具体定义日志的五个属性并实现Status接口中的方法，在具体的如ErrorStatus类实现StatusBase类，只是通过构造方法设置level区分不同的级别；
	+ StatusManager接口定义保存保存日志Status接口相关的方法接口，在子类BasicStatusManager中具体定义保存Status的集合并实现StatusManager中的方法
**使用实例**

ContextAware的子类GenericConfigurator：
```java
public final void doConfigure(File file) throws JoranException {
    FileInputStream fis = null;
    try {
        URL url = file.toURI().toURL();
        informContextOfURLUsedForConfiguration(getContext(), url);
        fis = new FileInputStream(file);
        doConfigure(fis, url.toExternalForm());
    } catch (IOException ioe) {
        String errMsg = "Could not open [" + file.getPath() + "].";
        addError(errMsg, ioe);
        throw new JoranException(errMsg, ioe);
    } finally {
        if (fis != null) {
            try {
                fis.close();
            } catch (java.io.IOException ioe) {
                String errMsg = "Could not close [" + file.getName() + "].";
                addError(errMsg, ioe);
                throw new JoranException(errMsg, ioe);
            }
        }
    }
}
```

ContextAware的子类SaxEventRecorder：
```java
public List<SaxEvent> recordEvents(InputSource inputSource) throws JoranException {
    SAXParser saxParser = buildSaxParser();
    try {
        saxParser.parse(inputSource, this);
        return saxEventList;
    } catch (IOException ie) {
        handleError("I/O error occurred while parsing xml file", ie);
    } catch (SAXException se) {
        // Exception added into StatusManager via Sax error handling. No need to add it again
        throw new JoranException("Problem parsing XML document. See previously reported errors.", se);
    } catch (Exception ex) {
        handleError("Unexpected exception while parsing XML document.", ex);
    }
    throw new IllegalStateException("This point can never be reached");
}

private void handleError(String errMsg, Throwable t) throws JoranException {
    addError(errMsg, t);
    throw new JoranException(errMsg, t);
}

public void error(SAXParseException spe) throws SAXException {
    addError(XML_PARSING + " - Parsing error on line " + spe.getLineNumber() + " and column " + spe.getColumnNumber());
    addError(spe.toString());
}

public void fatalError(SAXParseException spe) throws SAXException {
    addError(XML_PARSING + " - Parsing fatal error on line " + spe.getLineNumber() + " and column " + spe.getColumnNumber());
    addError(spe.toString());
}

public void warning(SAXParseException spe) throws SAXException {
    addWarn(XML_PARSING + " - Parsing warning on line " + spe.getLineNumber() + " and column " + spe.getColumnNumber(), spe);
}
```

ContextAware的子类SimpleRuleStore：
```java
public void addRule(ElementSelector elementSelector, String actionClassName) {
    Action action = null;
    try {
        action = (Action) OptionHelper.instantiateByClassName(actionClassName, Action.class, context);
    } catch (Exception e) {
        addError("Could not instantiate class [" + actionClassName + "]", e);
    }
    if (action != null) {
        addRule(elementSelector, action);
    }
}
```

ContextAware的子类InterpretationContext：
```
public void addInPlayListener(InPlayListener ipl) {
    if (listenerList.contains(ipl)) {
        addWarn("InPlayListener " + ipl + " has been already registered");
    } else {
        listenerList.add(ipl);
    }
}
```

## **6.问题及收获**

**问题**
	logback如何针对一个模块编写单元测试（如status-print，propertyAction，TrivialConfigurator等），它拆分的模块如何简化测试的难度，单元测试的一些技巧等。
	针对status-print的设计理念是什么，要解决那些问题，解决具体问题的相关代码，进行了那些性能优化、那些设计增强了灵活性、设计上的取舍（如确定status中应该具有的那6个标记日志的属性，日志级别只有info,warn,error三级这样可以最大限度的满足现有需求吗）。
	模块代码组织的包层级设计如何清晰明了及设计理念。
	对上下文对象Context在框架中如何使用，如何赋值和扩散（参考PropertyActionTest和TrivialConfiguratorTest测试对Context的处理，以及这样处理对测试带来的便利）。
	
**收获**
	根据模块的单元测试可以观察框架这个模块关注的重要功能点，辅助对模块源码的理解。
	把一个大的功能划分为功能相对独立的模块来理解，各个击破，这样画的类图和时序图更简洁有针对性，一目了然。
	blog对关键代码的理解，最好能带有代码，可以从其他优秀技术博文中借鉴思路。
	把日志封装到Status对象中，这样就可以实现日志的保存，实现根据指定的时间阈值过滤指定级别的日志是否存在，并据此作出接下来处理需求。如下：

```
public final void doConfigure(final InputSource inputSource) throws JoranException {

    long threshold = System.currentTimeMillis();
    // if (!ConfigurationWatchListUtil.wasConfigurationWatchListReset(context)) {
    // informContextOfURLUsedForConfiguration(getContext(), null);
    // }
    SaxEventRecorder recorder = new SaxEventRecorder(context);
    recorder.recordEvents(inputSource);
    doConfigure(recorder.saxEventList);
    // no exceptions a this level
    // 解析完文档，判断是否有Error级别的日志，据此输出解析是否成功
    StatusUtil statusUtil = new StatusUtil(context);
    if (statusUtil.noXMLParsingErrorsOccurred(threshold)) {
        addInfo("Registering current configuration as safe fallback point");
        registerSafeConfiguration(recorder.saxEventList);
    }
}
```