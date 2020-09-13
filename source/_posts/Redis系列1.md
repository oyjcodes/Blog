---
title: 挑战Redis系列(1)——协议
date: 2020-09-07 23:29:56
tags: 
- 数据库
categories: "Redis"  
---
```txt
RESP(REdis Serialization Protocol),也就是专门为redis设计的一套序列化协议. 这个协议其实在redis的1.2版本时就已经出现了,但是到了redis2.0才最终成为redis通讯协议的标准
```

## Redis通讯协议（TCP层）

特点 | 为什么
----|-----
二进制安全的 | 不需要处理从一个进程传输到另一个进程的批量数据，因为它使用前缀长度来传输批量数据。
可读性高| 简单
快速解析| 简单
易于实现| 简单

```txt
注意:此处概述的协议仅用于客户端 - 服务器通信。 Redis Cluster使用不同的二进制协议，以便在节点之间交换消息。
```

## RESP数据类型
Redis协议将传输的结构数据分为5种类型，第一个字节的符号来表示不同的数据类型,单元结束时统一加上回车换行符号 \r\n。

  类型 | 标识符 | 形式 | 例子
----|------|------|------------
简单字符串（simple string） | + | 放在第一个字节 | "+OK\r\n"
错误消息（error）| - | 放在第一个字节 | "-ERR unknown command 'foobar'\r\n"
长字符串（bulk string）<512M| $ | 放在第一个字节，后面跟字符串的长度 | "$0\r\n"   --$后面的0表示这是一个空字符串
整型数字（integer）| : | 放在第一个字节，后面跟整形的字符串 |  ":1000\r\n"
数组（arrays）| * | 放在第一个字节，后面跟数组的长度 | "*0\r\n"   --*后面的0表示表示空的数组

### 简单字符串

```txt
simple string 的第一个字节是个"+"(加号), 后面接着的是字符串的内容, 最后以CRLF(\r\n)结尾.例如:
"+hello world\r\n"
```
### 错误消息

```txt
 error其实和string是类似的, 但是RESP为了能让不同客户端把这种error和正常的返回结果区分开来对待 (例如redis返回error的话,就抛出异常),特意多设计了这个数据类型。error类型的第一个字节是"-"(减号), 后面接着的是错误的信息, 最后以CRLF(\r\n)结尾,例如:
 -ERR unknown command 'sets'\r\n"
```
### 长字符串

```txt
本质上也是字符串,跟普通字符串区分开来, 它的第一个字节是"$"(美元符号),紧接着是一个整数,表示字符串的字节数,字节数后面接一个CRLF. CRLF后面是字符串的内容, 最后以一个CRLF结尾. 例如:

"$0\r\n"   --$后面的0表示这是一个空字符串

"$-1\r\n"  -- $后面的-1表示这是一个null字符串,Null Bulk String要求客户端返回空对象,而不能简单地返回个空字符串

"$\r\n6ABCDEF\r\n"  -- ABCDEF是6个字节,所以$后面是6
```
### 整型数字
":29\r\n"

### 数组
```txt
第一个字节是"*"(星号), 紧接着后面是一个数字,表示这个数组的长度,数字后面是一个CRLF. 需要注意的是这个CRLF之后才是数组的真正内容, 而且数组内容可以是任意类型, 包括arrays和bulk string, 每个元素也要以CRLF结尾. 最后以CRLF(\r\n)结尾. 例如:

"*0\r\n"   --*后面的0表示表示空的数组

"*-1\r\n"  --*后面的-1表示表示是null数组

"*5\r\n     -- *5表示这是一个拥有5个元素的数组
+bar\r\n    -- 第1个元素是简单的字符串
-unknown command\r\n      -- 第2个元素是个异常
:3\r\n      -- 第3个元素是个整数
$3\r\n      -- 第4个元素是长度为3个字节的长字符串foo
foo\r\n     -- 第4个元素的内容
*3\r\n      -- 第5个元素又是个数组
:1\r\n      -- 第5个元素数组的第1元素
:2\r\n      -- 第5个元素数组的第2元素
:3\r\n      -- 第5个元素数组的第3元素
"
```
## request-response模型
```
 1. redis发送一个命令到服务端（一般组装成bulk string的数组）, 然后阻塞在socket.read()方法, 等待服务端的返回
 2. 服务端收到一个命令, 处理完成后将数据发送回去给客户端
```
redis的大部分命令都是使用这种request-response模型进行通讯, 除了以下两种特殊的情况
```
1. pipeline模式. 在pipeline模式下, 客户端可能会把多个命令收集在一起, 然后一并发送给服务端, 最后等待服务端把所有命令的执行响应一并发送回来
2. pub/sub, 发布订阅模式下, redis客户端只需要发送一次订阅命令
```
## 测试和验证
```java
/**
 * @title: RedisClient
 * @description: 撸一个简单的客户端
 */
public class RedisClient {

    private static Socket socket;
    private static OutputStream write;
    private static InputStream read;

    public static void main(String[] args) throws IOException {
        socket = new Socket("127.0.0.1",6379);
        write = socket.getOutputStream();
        read = socket.getInputStream();

        Scanner scan = new Scanner(System.in);

        // 判断是否还有输入
        while (scan.hasNextLine()) {
            String str = scan.nextLine();

            // 构造协议
            String commannd = buildCommand(str);

            System.out.println("发送命令为：\r\n" + commannd);

            String result = sendCommand(commannd);

            System.out.println("响应命令为：" + result);
        }

        scan.close();
    }

    /**
    　* @Description: 发送协议到客户端
    　* @param
    　* @return
     */
    private static String sendCommand(String command) throws IOException {
        write.write(command.getBytes());
        byte[] bytes = new byte[1024];
        read.read(bytes);

        return new String(bytes,"UTF-8");
    }

    /**
    　* @Description: 构建RESP协议
    　* @param
    　* @return
     */
    private static String buildCommand(String str) {
        if (!StringUtils.isEmpty(str)) {
            String[] strs = str.split(" ");

            StringBuilder builder = new StringBuilder();
            builder.append("*").append(strs.length).append("\r\n");
            for (String str1 : strs) {
                builder.append("$").append(str1.length()).append("\r\n");
                builder.append(str1).append("\r\n");
            }
            return builder.toString();
        }
        return null;
    }

}
```

## 运行结果
![avatar](1.png)<br>

![avatar](2.png)<br>

![avatar](3.png)<br>





