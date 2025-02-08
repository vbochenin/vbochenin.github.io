---
title: "Accidental logforging"
layout: post
author: vbochenin
tags:
  - java
  - security
categories:
  - dev-journal
---
## Problem we faced with

A few days ago, my mentee asked me why he can't see some log messages in the console. He was asked to add logging for incoming JMS message body to investigate issue we have during its processing.

The code in JMS messages consumer looks like this:

```java
  public void onMessage(TextMessage message) throws JMSException {
      log.info("Going to process message with id: {}", message.getJMSMessageID());
      try {
          processor.process(message.getText());
      } catch (Exception e) {
          log.error("Failed to process message: '{}' with id: {}", message.getText(), message.getJMSMessageID());
      }
  }
```

And the console output looks like this:

```bash
12:15:34.069 [main] DEBUG i.g.v.l.AccidentalLogforgingTest -- Going to process message with id: ce243af2-ef9a-4bc8-9160-39e8908916ab
    with headers: ce243af2-ef9a-4bc8-9160-39e8908916ab
```

The same output is for `cat` command as well. 

As you can see, the second line does not contain any mandatory log message attributes, such as time and log level, but only the tail of the message.

Also, it is only reproducible for a few messages.

So it looks like someone somewhere is truncating log messages and printing only the tail.

We've checked if there's a specific [logback](https://logback.qos.ch/) feature or setting responsible for this, but haven't found anything.

When we received the erroneous message from support, we found that it contained a '\r' character at the very end, followed by a few spaces.

```bash
"There is some message\r   "
```

So the message is printed, it is in the log file, but console processes [Carriage Return](https://en.wikipedia.org/wiki/Carriage_return) special characters during the printing.

The control character means to reset cursor position to the beginning of the line. 

It is also visible with `less` command (there is `^M` unprintable character)

```bash
12:15:34.069 [main] DEBUG i.g.v.l.AccidentalLogforgingTest -- Going to process message with id: ce243af2-ef9a-4bc8-9160-39e8908916ab
12:15:34.075 [main] ERROR i.g.v.l.AccidentalLogforgingTest -- Failed to process message: 'There is some message^M   ' with headers: ce243af2-ef9a-4bc8-9160-39e8908916ab
```

## How did we fix it

The problem is a well-known security issue called  [Logforging](https://owasp.org/www-community/attacks/Log_Injection). 

In short, if you log something from an untrusted source as is, you cannot rely on the correctness of your log messages. The problem is how to define which source is trustworthy. From one point of view, for the sake of simplicity, we can define any external source as untrusted, no matter if it is inside our closed environment boundaries or completely outside. But from another point of view, any additional processing of log message to escape control characters and injections is degrading performance and leads to loss of information about what exactly was input, which can be really significant during bug fixing.

In our case, we decided to escape characters because we need the solution only for a limited period of time until the original bug is discovered and fixed, and we don't log any input anywhere else:

```java
log.error("Failed to process message: '{}' with headers: {}", message.getText().replace("\r", "\\r"), message.getJMSMessageID());
 
```
 

An alternative solution could be to use the  [org.owasp.security-logging-logback]([https://mvnrepository.com/artifact/org.owasp/security-logging-logback](https://mvnrepository.com/artifact/org.owasp/security-logging-logback)) library to add a conversion rule to the logger configuration and get escape for multiple control characters out of the box.

```xml
<configuration>
    <conversionRule conversionWord="crlf"
                    converterClass="org.owasp.security.logging.mask.NLFConverter" />
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} -%kvp- %crlf(%msg) %n</pattern>
        </encoder>
    </appender>   
```

So the configuration produces following output:

```bash
16:46:02.019 [main] DEBUG i.g.v.l.AccidentalLogforgingTest -- Going to process message with id: a15c28fa-df09-48e1-83d2-0203acf479e1 
16:46:02.021 [main] ERROR i.g.v.l.AccidentalLogforgingTest -- Failed to process message: 'There is some message\r   ' with headers: a15c28fa-df09-48e1-83d2-0203acf479e1 
```

_You may find the code in [GitHub](https://github.com/vbochenin/code.vbochenin.github.io/tree/main/accidental-logforging)._ 
