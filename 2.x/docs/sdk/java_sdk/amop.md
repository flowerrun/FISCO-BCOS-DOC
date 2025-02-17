# AMOP 功能

标签：``java-sdk`` ``AMOP`` ``链上信使`` 

----
Java SDK支持[链上信使协议AMOP（Advanced Messages Onchain Protocol）](../../manual/amop_protocol.html)，用户可以通过AMOP协议与其它机构互传消息。



## 1. 配置方法

AMOP有两种话题模式普通话题和私有话题。任何一个订阅了某普通话题的订阅者都能收到该话题相关的推送消息。但在某些情况下，发送者只希望特定的接收者能接收到消息，不希望无关的接收者能任意的监听此话题，这时就需要使用AMOP私有话题了。

AMOP私有话题的特别之处在于，SDK间需要进行了身份认证，认证通过的订阅者才可以收到该话题的消息。身份认证的原理是，首先由发送方生成一个随机数，订阅方用私钥对随机数签名，发送方用所配置的公钥验证这个签名来确定对方是否是自己指定的订阅方。因此，一个成功的私有话题通道的建立需要（1）消息发送者需要设置指定的订阅者的公钥；（2）订阅方也需要设置能证明自己身份的私钥。

当用户需要订阅私有话题，或者作为消息发布者配置一个私有话题时，可用配置文件进行配置。但AMOP的配置不是必须项，私有话题的订阅和设置，还可以通过调用AMOP的接口实现。以下是AMOP的配置示例，是``test/resource/config-example.toml``配置文件中的一部分。

```eval_rst
.. important::
    注意：
    1. AMOP私有话题目前只支持非国密算法。请使用 `生成公私钥脚本 <./account.md>_ 生成非国密公私钥文件.
    2. 如果私有话题的发送者和订阅者连接了同一个节点，则被视为是同一个组织的两个SDK，这两个不需要进行私有话题认证流程即可通过该私有话题通信。
    3. 如果私有话题的两个发送者A1和A2连接了同一个节点Node1，该私有话题的订阅者B连接了Node2，Node1和Node2相连。若A1已经和B完成了私有话题的认证，则A2可以不经过认证，向B发送私有话题消息，因为同机构的A1已经对订阅者B的身份进行了验证。
```

```toml
# AMOP configuration
# You can use following two methods to configure as a private topic message sender or subscriber.
# Usually, the public key and private key is generated by subscriber.
# Message sender receive public key from topic subscriber then make configuration.
# But, please do not config as both the message sender and the subscriber of one private topic, or you may send the message to yourself.

# Configure a private topic as a topic message sender.
# [[amop]]
# topicName = "PrivateTopic"
# publicKeys = [ "conf/amop/consumer_public_key_1.pem" ]    # Public keys of the nodes that you want to send AMOP message of this topic to.

# Configure a private topic as a topic subscriber.
# [[amop]]
# topicName = "PrivateTopic"
# privateKey = "conf/amop/consumer_private_key.p12"         # Your private key that used to subscriber verification.
# password = "123456"
```

配置详解：

1. 配置一个私有话题（作为消息发布者）

   * 需要在配置文件中新建一个``[[amop]]``节点。
   * 并配置话题名称``topicName = "PrivateTopic"``
   * 消息订阅方的公钥列表``publicKeys = [ "conf/amop/consumer_public_key_1.pem" ]`` ，即您想指定的接收对象的公钥，这个公钥必须与某个订阅方的私钥所匹配。

   ```toml
   [[amop]]
   topicName = "PrivateTopic"
   publicKeys = [ "conf/amop/consumer_public_key_1.pem" ]
   ```

2. 订阅一个私有话题（作为订阅者）

   * 需要在配置文件中新建一个``[[amop]]``节点。
   * 并配置话题名称``topicName = "PrivateTopic"``
   * 证明订阅方身份的私钥``privateKey = "conf/amop/consumer_private_key.p12"`` 
   * 该私钥的密码``password = "123456"``

   ```toml
   [[amop]]
   topicName = "PrivateTopic"
   privateKey = "conf/amop/consumer_private_key.p12"
   password = "123456"
   ```
  
   

## 2. 接口说明

AMOP模块的接口类可参考文件java-sdk中的``sdk-amop/src/main/org/fisco/bcos/sdk/amop/Amop.java``文件，其中主要包含以下几个接口：

#### subscribeTopic

订阅一个普通话题

**参数：**

* topicName: 话题名称。类型：``String``。
* callback: 处理该话题消息的函数，当收到该话题相关消息时，会被调用。类型：``AmopCallback``。

**例子：**

```java
// 初始化java SDK, 获得Amop对象
BcosSDK sdk = new BcosSDK("config-example.toml");
Amop amop = sdk.getAmop();

// 定义一个Callback，重写receiveAmopMsg方法，定义收到消息后的处理流程。
AmopCallback cb = new AmopCallback() {
  @Override
  public byte[] receiveAmopMsg(AmopMsgIn msg) {
    // 你可以在这里写收到消息后的处理逻辑。
    System.out.println("Received msg, content:" + new String(msg.getContent()));
    return "Yes, I received.".getBytes();
  }
};
// 订阅话题
amop.subscribeTopic("MyTopic", cb);
```



#### subscribePrivateTopic

订阅一个私有话题

**参数：**

* topicName: 话题名称。类型：``String``。
* privateKeyTool：证明订阅者身份的私钥信息。类型：``KeyTool``。
* callback: 处理该话题消息的函数，当收到该话题相关消息时，会被调用。类型：``AmopCallback``。

**例子：**

```java
// 初始化java SDK, 获得Amop对象
BcosSDK sdk = new BcosSDK("config-example.toml");
Amop amop = sdk.getAmop();

// 定义一个Callback，重写receiveAmopMsg方法，定义收到消息后的处理流程。
AmopCallback cb = new AmopCallback() {
  @Override
  public byte[] receiveAmopMsg(AmopMsgIn msg) {
    // 你可以在这里写收到消息后的处理逻辑。
    System.out.println("Received msg, content:" + new String(msg.getContent()));
    return "Yes, I received.".getBytes();
  }
};

// 加载私钥
KeyTool km = new P12KeyStore("private_key.p12", "12s230");

// 订阅话题
amop.subscribePrivateTopics("MyPrivateTopic", km, cb);
```



#### publishPrivateTopic

作为消息发送者设置一个私有话题。

**参数：**

* topicName: 话题名称。类型：``String``。
* publicKeyTools：指定订阅者的公钥列表。类型：``List<KeyTool>``。

**例子**：

```java
// 初始化java SDK, 获得Amop对象
BcosSDK sdk = new BcosSDK("config-example.toml");
Amop amop = sdk.getAmop();

// 加载指定订阅者的私钥列表
List<KeyTool> publicKeyList = new ArrayList<>();
KeyTool pubKey1 = new PEMKeyStore("target_subscriber1_public_key.pem");
KeyTool pubKey2 = new PEMKeyStore("target_subscriber2_public_key.pem");
publicKeyList.add(pubKey1);
publicKeyList.add(pubKey2);

// 设置一个私有话题
amop.publishPrivateTopic("MyPrivateTopic", publicKeyList);
```



#### unsubscribeTopic

取消一个话题的订阅。

**参数：**

* topicName: 话题名称。类型：``String``。

**例子：**

```java
// 初始化java SDK, 获得Amop对象
BcosSDK sdk = new BcosSDK("config-example.toml");
Amop amop = sdk.getAmop();

// 订阅“MyTopic”，过程省略

// 取消订阅
amop.unsubscribeTopic("MyTopic");
```



#### sendAmopMsg

以单播的方式发送AMOP消息

**参数：**

* content: 消息内容。类型：``AmopMsgOut``。
* callback: 回调函数。类型：``ResponseCallback``。

**例子：**

```java
// 初始化java SDK, 获得Amop对象
BcosSDK sdk = new BcosSDK("config-example.toml");
Amop amop = sdk.getAmop();

// 构造内容
AmopMsgOut out = new AmopMsgOut();
out.setTopic("MyTopic");
out.setType(TopicType.NORMAL_TOPIC);
out.setContent("some content here".getBytes());
out.setTimeout(6000);

// 构造Callback
ResponseCallback cb =
  new ResponseCallback() {
  @Override
  public void onResponse(Response response) {
		// 你可以在这里写收到回复的处理逻辑。
    System.out.println(
      "Get response, { errorCode:"
      + response.getErrorCode()
      + " error:"
      + response.getErrorMessage()
      + " seq:"
      + response.getMessageID()
      + " content:"
      + new String(response.getContentBytes())
      + " }");
  }
};

// 发送消息
amop.sendAmopMsg(out, cb);
```



#### broadcastAmopMsg

以广播的方式发送AMOP消息

**参数：**

* content: 消息内容。类型：``AmopMsgOut``。

**例子：**

```java
// 初始化java SDK, 获得Amop对象
BcosSDK sdk = new BcosSDK("config-example.toml");
Amop amop = sdk.getAmop();

// 构造内容
AmopMsgOut out = new AmopMsgOut();
out.setTopic("MyTopic");
out.setType(TopicType.NORMAL_TOPIC);
out.setContent(content.getBytes());
out.setTimeout(6000);

// 发送消息
amop.broadcastAmopMsg(out);
```



#### getSubTopics

获取SDK当前订阅的话题

**参数：**

无

**返回值：**

* 订阅的话题集合。类型：``Set<String>``。

**例子：**

```java
// 初始化java SDK, 获得Amop对象
BcosSDK sdk = new BcosSDK("config-example.toml");
Amop amop = sdk.getAmop();

Set<String> topics = amop.getSubTopics();
```



#### start

启动amop功能，SDK初始化后默认启动。

**例子：**

```java
// 初始化java SDK, 获得Amop对象
BcosSDK sdk = new BcosSDK("config-example.toml");
Amop amop = sdk.getAmop();

amop.stop()
  
// 在需要的时候
amop.start()
```



#### stop

停止AMOP功能。

**例子：**

```java
// 初始化java SDK, 获得Amop对象
BcosSDK sdk = new BcosSDK("config-example.toml");
Amop amop = sdk.getAmop();

// 停止AMOP
amop.stop()
```



## 3. 示例

更多的示例请看[java-sdk-demo](https://github.com/FISCO-BCOS/java-sdk-demo)项目源码``java-sdk-demo/src/main/java/org/fisco/bcos/sdk/demo/amop/tool``下的代码示范，链接：[java-sdk-demo GitHub链接](https://github.com/FISCO-BCOS/java-sdk-demo)，[java-sdk-demo Gitee链接](https://gitee.com/FISCO-BCOS/java-sdk-demo)。

* 普通话题代码示例：

  * AmopPublisher
    输入： 话题名（TopicName），是否多播（isBroadcast, true 为多播，false为单拨），内容（Content），发送的数量（Count）
    功能：发送普通话题消息、广播普通话题消息
  * AmopPublisherFile
    输入： 话题名（TopicName），是否多播（isBroadcast），文件名（FileName），发送的数量（Count）
    功能：发送普通话题文件、广播普通话题文件
  * AmopSubscriber
    输入：话题名（TopicName）
    默认：订阅了一个名为“Test”的普通话题
    功能：订阅一个普通话题

* 私有话题代码示例：

  * AmopSubscriberPrivate

    输入：话题名（TopicName），私钥文件名（Filename），密码（password）
    默认：订阅了一个名为“PrivTest”的私有话题
    功能：订阅一个私有话题

  * AmopPublisherPrivate

    输入： 话题名（TopicName），公钥文件名1（Filename），公钥文件名2（Filename），是否多播（isBroadcast），内容（Content），发送的数量（Count）
    功能：配置私有话题发送，发送私有话题消息，广播私有话题消息

  * AmopPublisherPrivateFile

    输入：话题名（TopicName），公钥文件名1（Filename），公钥文件名2（Filename，如无则输入null），是否多播（isBroadcast），文件名（FileName），发送的数量（Count）
    测试功能：发送私有话题文件、广播普通话题文件
    

**示例讲解**

* 配置。

  Java SDK支持动态订阅话题。也可以在配置文件中配置固定的私有话题。

  订阅者配置例子：[java-sdk-demo](https://github.com/FISCO-BCOS/java-sdk-demo)项目源码``java-sdk-demo/src/resources/amop/config-subscriber-for-test.toml``

  ```toml
  [network]
  # The peer list to connect
  peers=["127.0.0.1:20202", "127.0.0.1:20203"]
  
  # Configure a private topic as a topic message sender.
  [[amop]]
  topicName = "privTopic"
  # Your private key that used to subscriber verification.
  privateKey = "conf/amop/consumer_private_key.p12"
  password = "123456"
  ```

  注意，订阅方通过这种方法配置的话题，需要在程序中设定一个默认的回调函数，否则，接收消息时会因为找不到回调函数而报错。

  发送者配置例子：[java-sdk-demo](https://github.com/FISCO-BCOS/java-sdk-demo)项目源码``java-sdk-demo/src/resources/amop/config-publisher-for-test.toml``

  ```toml
  [network]
  # The peer list to connect
  peers=["127.0.0.1:20200", "127.0.0.1:20201"]
  
  # Configure a "need verify AMOP topic" as a topic message sender.
  [[amop]]
  topicName = "privTopic"
  # Public keys of the nodes that you want to send AMOP message of this topic to.
  publicKeys = [ "conf/amop/consumer_public_key_1.pem"]
  ```

  发送者所配置的公钥是从订阅者那里获取的，与订阅者的私钥是成对的。这样发送者可以通过私有话题"privTopic"向订阅者发送消息。

* 公有话题订阅和发送

  订阅者代码例子：``src/main/java/org/fisco/bcos/sdk/demo/amop/tool/AmopSubscriber.java``

  ```java
  package org.fisco.bcos.sdk.demo.amop.tool;
  
  import java.net.URL;
  import org.fisco.bcos.sdk.BcosSDK;
  import org.fisco.bcos.sdk.amop.Amop;
  import org.fisco.bcos.sdk.amop.AmopCallback;
  
  public class AmopSubscriber {
  
      public static void main(String[] args) throws Exception {
          URL configUrl =
                  AmopSubscriber.class
                          .getClassLoader()
                          .getResource("amop/config-subscriber-for-test.toml");
          if (args.length < 1) {
              System.out.println("Param: topic");
              return;
          }
          String topic = args[0];
          // Construct a BcosSDK instance
          BcosSDK sdk = BcosSDK.build(configUrl.getPath());
          
          // Get the amop module instance
          Amop amop = sdk.getAmop();
          
          // Set callback
          AmopCallback cb = new DemoAmopCallback();
          // Set a default callback
          amop.setCallback(cb);
          // Subscriber a normal topic
          amop.subscribeTopic(topic, cb);
          System.out.println("Start test");
      }
  }
  ```

  回调函数例子: ``src/main/java/org/fisco/bcos/sdk/demo/amop/tool/DemoAmopCallback.java``

  ```java
  package org.fisco.bcos.sdk.demo.amop.tool;
  
  import static org.fisco.bcos.sdk.utils.ByteUtils.byteArrayToInt;
  
  import java.io.BufferedOutputStream;
  import java.io.File;
  import java.io.FileOutputStream;
  import java.io.IOException;
  import java.util.Arrays;
  import org.fisco.bcos.sdk.amop.AmopCallback;
  import org.fisco.bcos.sdk.amop.topic.AmopMsgIn;
  import org.fisco.bcos.sdk.model.MsgType;
  import org.slf4j.Logger;
  import org.slf4j.LoggerFactory;
  
  public class DemoAmopCallback extends AmopCallback {
      private static Logger logger = LoggerFactory.getLogger(DemoAmopCallback.class);
  
      @Override
      public byte[] receiveAmopMsg(AmopMsgIn msg) {
          if (msg.getContent().length > 8) {
              byte[] content = msg.getContent();
              byte[] byteflag = subbytes(content, 0, 4);
              int flag = byteArrayToInt(byteflag);
              if (flag == -128) {
                  // Received a file.
                  byte[] bytelength = subbytes(content, 4, 4);
                  int length = byteArrayToInt(bytelength);
                  byte[] bytefilename = subbytes(content, 8, length);
                  String filename = new String(bytefilename);
                  System.out.println(
                          "Step 2:Receive file, filename length:"
                                  + length
                                  + " filename binary:"
                                  + Arrays.toString(bytefilename)
                                  + " filename:"
                                  + filename);
  
                  int contentlength = content.length - 8 - filename.length();
                  byte[] fileContent = subbytes(content, 8 + filename.length(), contentlength);
                  getFileFromBytes(fileContent, filename);
                  System.out.println("|---save file:" + filename + " success");
                  byte[] responseData = "Yes, I received!".getBytes();
                  if (msg.getType() == (short) MsgType.AMOP_REQUEST.getType()) {
                      System.out.println("|---response:" + new String(responseData));
                  }
                  return responseData;
              }
          }
          
          
          byte[] responseData = "Yes, I received!".getBytes();
          // Print receive amop message
          System.out.println(
                  "Step 2:Receive msg, topic:"
                          + msg.getTopic()
                          + " content:"
                          + new String(msg.getContent()));
          if (msg.getType() == (short) MsgType.AMOP_REQUEST.getType()) {
              System.out.println("|---response:" + new String(responseData));
          }
          // Response to the message sender
          return responseData;
      }
  
      public static byte[] subbytes(byte[] src, int begin, int count) {
          byte[] bs = new byte[count];
          System.arraycopy(src, begin, bs, 0, count);
          return bs;
      }
  
      public static void getFileFromBytes(byte[] b, String outputFile) {
          File ret = null;
          BufferedOutputStream stream = null;
          FileOutputStream fstream = null;
          try {
              ret = new File(outputFile);
              fstream = new FileOutputStream(ret);
              stream = new BufferedOutputStream(fstream);
              stream.write(b);
          } catch (Exception e) {
              logger.error(" write exception, message: {}", e.getMessage());
          } finally {
              if (stream != null) {
                  try {
                      stream.close();
                  } catch (IOException e) {
                      logger.error(" close exception, message: {}", e.getMessage());
                  }
              }
  
              if (fstream != null) {
                  try {
                      fstream.close();
                  } catch (IOException e) {
                      logger.error(" close exception, message: {}", e.getMessage());
                  }
              }
          }
      }
  }
  ```

  发送方使用例子：``src/main/java/org/fisco/bcos/sdk/demo/amop/tool/AmopPublisher.java``

  ```java
  package org.fisco.bcos.sdk.demo.amop.tool;
  
  import org.fisco.bcos.sdk.BcosSDK;
  import org.fisco.bcos.sdk.amop.Amop;
  import org.fisco.bcos.sdk.amop.AmopMsgOut;
  import org.fisco.bcos.sdk.amop.topic.TopicType;
  
  public class AmopPublisher {
      private static final int parameterNum = 4;
      private static String publisherFile =
              AmopPublisher.class
                      .getClassLoader()
                      .getResource("amop/config-publisher-for-test.toml")
                      .getPath();
  
      /**
       * @param args topicName, isBroadcast, Content(Content you want to send out), Count(how many msg
       *     you want to send out)
       * @throws Exception AMOP publish exceptioned
       */
      public static void main(String[] args) throws Exception {
          if (args.length < parameterNum) {
              System.out.println("param: target topic total number of request");
              return;
          }
          String topicName = args[0];
          Boolean isBroadcast = Boolean.valueOf(args[1]);
          String content = args[2];
          Integer count = Integer.parseInt(args[3]);
          BcosSDK sdk = BcosSDK.build(publisherFile);
          Amop amop = sdk.getAmop();
  
          System.out.println("3s ...");
          Thread.sleep(1000);
          System.out.println("2s ...");
          Thread.sleep(1000);
          System.out.println("1s ...");
          Thread.sleep(1000);
  
          System.out.println("start test");
          System.out.println("===================================================================");
  
          for (Integer i = 0; i < count; ++i) {
              Thread.sleep(2000);
              AmopMsgOut out = new AmopMsgOut();
              out.setType(TopicType.NORMAL_TOPIC);
              out.setContent(content.getBytes());
              out.setTimeout(6000);
              out.setTopic(topicName);
              DemoAmopResponseCallback cb = new DemoAmopResponseCallback();
              if (isBroadcast) {
                	// send out amop message by broad cast
                  amop.broadcastAmopMsg(out);
                  System.out.println(
                          "Step 1: Send out msg by broadcast, topic:"
                                  + out.getTopic()
                                  + " content:"
                                  + new String(out.getContent()));
              } else {
                	// send out amop message
                  amop.sendAmopMsg(out, cb);
                  System.out.println(
                          "Step 1: Send out msg, topic:"
                                  + out.getTopic()
                                  + " content:"
                                  + new String(out.getContent()));
              }
          }
      }
  }
  
  ```

  发送方接收回包的函数例子：``src/main/java/org/fisco/bcos/sdk/demo/amop/tool/DemoAmopResponseCallback.java``

  ```java
  package org.fisco.bcos.sdk.demo.amop.tool;
  
  import org.fisco.bcos.sdk.amop.AmopResponse;
  import org.fisco.bcos.sdk.amop.AmopResponseCallback;
  
  public class DemoAmopResponseCallback extends AmopResponseCallback {
  
      @Override
      public void onResponse(AmopResponse response) {
        	// 当出现102超时错误时，打印该错误
          if (response.getErrorCode() == 102) {
              System.out.println(
                      "Step 3: Timeout, maybe your file is too large or your gave a short timeout.");
          } else {
         			// 收到正常的回包
              if (response.getAmopMsgIn() != null) {
                  System.out.println(
                          "Step 3:Get response, { errorCode:"
                                  + response.getErrorCode()
                                  + " error:"
                                  + response.getErrorMessage()
                                  + " seq:"
                                  + response.getMessageID()
                                  + " content:"
                                  + new String(response.getAmopMsgIn().getContent())
                                  + " }");
              } else {
                // 收到其它错误
                  System.out.println(
                          "Step 3:Get response, { errorCode:"
                                  + response.getErrorCode()
                                  + " error:"
                                  + response.getErrorMessage()
                                  + " seq:"
                                  + response.getMessageID());
              }
          }
      }
  }
  ```

* 私有话题的发送和订阅:

  订阅一个私有话题例子：``src/main/java/org/fisco/bcos/sdk/demo/amop/tool/AmopSubscriberPrivate.java``

  ```java
  package org.fisco.bcos.sdk.demo.amop.tool;
  
  import org.fisco.bcos.sdk.BcosSDK;
  import org.fisco.bcos.sdk.amop.Amop;
  import org.fisco.bcos.sdk.amop.AmopCallback;
  import org.fisco.bcos.sdk.crypto.keystore.KeyTool;
  import org.fisco.bcos.sdk.crypto.keystore.P12KeyStore;
  import org.fisco.bcos.sdk.crypto.keystore.PEMKeyStore;
  
  public class AmopSubscriberPrivate {
      private static String subscriberConfigFile =
              AmopSubscriberPrivate.class
                      .getClassLoader()
                      .getResource("amop/config-subscriber-for-test.toml")
                      .getPath();
  
      /**
       * @param args topic, privateKeyFile, password(Option)
       * @throws Exception AMOP exceptioned
       */
      public static void main(String[] args) throws Exception {
          if (args.length < 2) {
              System.out.println("Param: topic, privateKeyFile, password");
              return;
          }
          String topic = args[0];
          String privateKeyFile = args[1];
          BcosSDK sdk = BcosSDK.build(subscriberConfigFile);
          Amop amop = sdk.getAmop();
          AmopCallback cb = new DemoAmopCallback();
  
          System.out.println("Start test");
          amop.setCallback(cb);
          
          // Read a private key file
          KeyTool km;
          if (privateKeyFile.endsWith("p12")) {
              String password = args[2];
              km = new P12KeyStore(privateKeyFile, password);
          } else {
              km = new PEMKeyStore(privateKeyFile);
          }
          
          // Subscriber a private topic.
          amop.subscribePrivateTopics(topic, km, cb);
      }
  }
  ```

  发送私有话题消息例子：

  ```java
  package org.fisco.bcos.sdk.demo.amop.tool;
  
  import java.util.ArrayList;
  import java.util.List;
  import org.fisco.bcos.sdk.BcosSDK;
  import org.fisco.bcos.sdk.amop.Amop;
  import org.fisco.bcos.sdk.amop.AmopMsgOut;
  import org.fisco.bcos.sdk.amop.topic.TopicType;
  import org.fisco.bcos.sdk.crypto.keystore.KeyTool;
  import org.fisco.bcos.sdk.crypto.keystore.PEMKeyStore;
  
  public class AmopPublisherPrivate {
      private static final int parameterNum = 6;
      private static String publisherFile =
              AmopPublisherPrivate.class
                      .getClassLoader()
                      .getResource("amop/config-publisher-for-test.toml")
                      .getPath();
  
      /**
       * @param args topicName, pubKey1, pubKey2, isBroadcast: true/false, content, count. if only one
       *     public key please fill pubKey2 with null
       * @throws Exception AMOP exceptioned
       */
      public static void main(String[] args) throws Exception {
          if (args.length < parameterNum) {
              System.out.println(
                      "param: opicName, pubKey1, pubKey2, isBroadcast: true/false, content, count");
              return;
          }
          String topicName = args[0];
          String pubkey1 = args[1];
          String pubkey2 = args[2];
          Boolean isBroadcast = Boolean.valueOf(args[3]);
          String content = args[4];
          Integer count = Integer.parseInt(args[5]);
          BcosSDK sdk = BcosSDK.build(publisherFile);
          Amop amop = sdk.getAmop();
  
          System.out.println("3s ...");
          Thread.sleep(1000);
          System.out.println("2s ...");
          Thread.sleep(1000);
          System.out.println("1s ...");
          Thread.sleep(1000);
  
          System.out.println("start test");
          System.out.println("===================================================================");
          System.out.println("set up private topic");
          List<KeyTool> keyToolList = new ArrayList<>();
          
          // Read public key files.
          KeyTool keyTool = new PEMKeyStore(pubkey1);
          keyToolList.add(keyTool);
          if (!pubkey2.equals("null")) {
              KeyTool keyTool1 = new PEMKeyStore(pubkey2);
              keyToolList.add(keyTool1);
          }
          // Publish a private topic
          amop.publishPrivateTopic(topicName, keyToolList);
          System.out.println("wait until finish private topic verify");
          System.out.println("3s ...");
          Thread.sleep(1000);
          System.out.println("2s ...");
          Thread.sleep(1000);
          System.out.println("1s ...");
          Thread.sleep(1000);
  
          for (Integer i = 0; i < count; ++i) {
              Thread.sleep(2000);
              AmopMsgOut out = new AmopMsgOut();
              // It is a private topic.
              out.setType(TopicType.PRIVATE_TOPIC);
              out.setContent(content.getBytes());
              out.setTimeout(6000);
              out.setTopic(topicName);
              DemoAmopResponseCallback cb = new DemoAmopResponseCallback();
              if (isBroadcast) {
                  // Send out message by broadcast
                  amop.broadcastAmopMsg(out);
                  System.out.println(
                          "Step 1: Send out msg by broadcast, topic:"
                                  + out.getTopic()
                                  + " content:"
                                  + new String(out.getContent()));
              } else {
                  // Send out amop message
                  amop.sendAmopMsg(out, cb);
                  System.out.println(
                          "Step 1: Send out msg, topic:"
                                  + out.getTopic()
                                  + " content:"
                                  + new String(out.getContent()));
              }
          }
      }
  }
  ```

接下来，可以根据下一节的方法来试用这些AMOP的Demo。



## 4. 快速试用AMOP

### 第一步：下载项目

```bash
mkdir -p ~/fisco && cd ~/fisco
# 获取java-sdk代码
git clone https://github.com/FISCO-BCOS/java-sdk-demo

# 若因为网络问题导致长时间拉取失败，请尝试以下命令：
git clone https://gitee.com/FISCO-BCOS/java-sdk-demo

cd java-sdk-demo
# 切换到2.0版本
git checkout main-2.0
# 构建项目
bash gradlew build
```

### 第二步：搭建FISCO BCOS区块链网络

方法一：如果您使用的操作系统为Linux或Darwin，可以用将以下脚本保存到java-sdk的目录下，命名为initEnv.sh，并运行该脚本。

initEnv.sh

```bash
#!/bin/bash
download_tassl()
{
  mkdir -p ~/.fisco/
  if [ "$(uname)" == "Darwin" ];then
    curl -LO https://github.com/FISCO-BCOS/LargeFiles/raw/master/tools/tassl_mac.tar.gz
    mv tassl_mac.tar.gz ~/.fisco/tassl.tar.gz
  else
    curl -LO https://github.com/FISCO-BCOS/LargeFiles/raw/master/tools/tassl.tar.gz
    mv tassl.tar.gz ~/.fisco/tassl.tar.gz
  fi
  tar -xvf ~/.fisco/tassl.tar.gz
}

build_node()
{
  local node_type="${1}"
  if [ "${node_type}" == "sm" ];then
      ./build_chain.sh -l 127.0.0.1:4 -g
      sed_cmd=$(get_sed_cmd)
      $sed_cmd 's/sm_crypto_channel=false/sm_crypto_channel=true/g' nodes/127.0.0.1/node*/config.ini
  else
      ./build_chain.sh -l 127.0.0.1:4
  fi
  ./nodes/127.0.0.1/fisco-bcos -v
  ./nodes/127.0.0.1/start_all.sh
}
download_build_chain()
{
  tag=$(curl -sS "https://gitee.com/api/v5/repos/FISCO-BCOS/FISCO-BCOS/tags" | grep -oe "\"name\":\"v[2-9]*\.[0-9]*\.[0-9]*\"" | cut -d \" -f 4 | sort -V | tail -n 1)
  LOG_INFO "--- current tag: $tag"
  curl -LO "https://github.com/FISCO-BCOS/FISCO-BCOS/releases/download/${tag}/build_chain.sh" && chmod u+x build_chain.sh
}
prepare_environment()
{
  ## prepare resources for amop demo
  bash gradlew build -x test
  cp -r nodes/127.0.0.1/sdk/* sdk-demo/dist/conf
}

download_tassl
./gradlew build -x test
download_build_chain
build_node
prepare_environment
```

运行该文件：

```bash
# 更改文件权限
chmod 777 initEnv.sh     
# 运行initEnv.sh文件
./initEnv.sh
```

运行完成后，终端显示4个节点已经启动，demo环境就已经准备好了。

```bash
FISCO-BCOS Version : 2.6.0
Build Time         : 20200814 09:04:17
Build Type         : Darwin/appleclang/RelWithDebInfo
Git Branch         : HEAD
Git Commit Hash    : e4a5ef2ef64d1943fccc4ebc61467a91779fb1c0
try to start node0
try to start node1
try to start node2
try to start node3
 node1 start successfully
 node3 start successfully
 node0 start successfully
 node2 start successfully
```

方法二：您也可以根据[指引](../../installation.html#fisco-bcos)搭建FISCO BCOS区块链网络。然后进行以下操作

```cmd
# 当前目录为java-sdk,构建项目
gradlew.bat build -x test
```



### 第三步：配置

* 复制证书：将你搭建FISCO BCOS网络节点``nodes/${ip}/sdk/`` 目录下的证书复制到``java-sdk-demo/dist/conf``目录下。

* 修改配置：如果您采用的是方法一搭建的网络，则无需修改配置。如果您采用方法二搭建区块链，需要修改订阅者配置文件``java-sdk-demo/dist/conf/amop/config-subscriber-for-test.toml``，和发送者配置文件``java-sdk-demo/dist/conf/amop/config-publisher-for-test.toml``，修改配置文件中的节点信息。 注意：订阅者和发送者最好不连相同节点，如果连接了相同节点，则会被认为是同一个组织下的成员，私有话题无需认证即可通讯。

  

### 第四步：运行Demo

#### 公有话题Demo

新打开一个终端，下载java-sdk-demo的代码，并build。

```bash
cd ~/fisco
# 获取java-sdk-demo代码
git clone https://github.com/FISCO-BCOS/java-sdk-demo

# 若因为网络问题导致长时间拉取失败，请尝试以下命令：
git clone https://gitee.com/FISCO-BCOS/java-sdk-demo

cd java-sdk-demo
# 切换到2.0版本
git checkout main-2.0
# build项目
bash gradlew build
```

**运行订阅者：**

```bash
# 进入java-sdk-demo/dist目录
cd dist 
# 使用第三节中所描述的工具
# 我们订阅名为”testTopic“的话题
java -cp "apps/*:lib/*:conf/" org.fisco.bcos.sdk.demo.amop.tool.AmopSubscriber testTopic
```

订阅方的终端输出

```bash
Start test
```

然后，运行发送者Demo

**单播消息**：

```bash
# 调用AmopPublisher发送AMOP消息
# 话题名：testTopic，是否广播：false(即使用单播)，内容：Tell you something， 发送次数：2次
java -cp "apps/*:lib/*:conf/" org.fisco.bcos.sdk.demo.amop.tool.AmopPublisher testTopic false "Tell you something" 2
```

终端输出：

```bash
3s ...
2s ...
1s ...
start test
===================================================================
Step 1: Send out msg, topic:testTopic content:Tell you something
Step 3:Get response, { errorCode:0 error:null seq:3fa95b760f7f48ddbdf1216a48f361e0 content:Yes, I received! }
Step 1: Send out msg, topic:testTopic content:Tell you something
Step 3:Get response, { errorCode:0 error:null seq:2bc754b1d8844445a4cc2af226fbaa58 content:Yes, I received! }
```

同时，返回到话题订阅者的终端，发现终端输出：

```bash
Step 2:Receive msg, topic:testTopic content:Tell you something
|---response:Yes, I received!
Step 2:Receive msg, topic:testTopic content:Tell you something
|---response:Yes, I received!
```

**广播消息**：

```bash
# 调用AmopPublisher发送AMOP消息
# 话题名：testTopic，是否广播：false(即使用单播)，内容：Tell you something， 发送次数：1次
java -cp "apps/*:lib/*:conf/" org.fisco.bcos.sdk.demo.amop.tool.AmopPublisher testTopic true "Tell you something" 1
```

终端的输出

```bash
3s ...
2s ...
1s ...
start test
===================================================================
Step 1: Send out msg by broadcast, topic:testTopic content:Tell you something
```

同时，返回到话题订阅者的终端，发现终端输出：

```java
Start test
Step 2:Receive msg, topic:testTopic content:Tell you something
Step 2:Receive msg, topic:testTopic content:Tell you something
Step 2:Receive msg, topic:testTopic content:Tell you something
Step 2:Receive msg, topic:testTopic content:Tell you something
```

注意：

1. 广播消息没有回包。

2. 接收方可能收到多条重复的广播信息。比如，上述例子中，网络中总共有4个节点，发送发连接节点1和2，接收方连接3和4。因此，广播的时候存在四条路径[发送方 -> 节点1 -> 节点3 -> 接收方]，[发送方 -> 节点1 -> 节点4 -> 接收方]，[发送方 -> 节点2 -> 节点3 -> 接收方]，[发送方 -> 节点2 -> 节点4 -> 接收方]，则接收方SDK收到了4条信息。

**发送文件**：

```bash
# 调用AmopPublisherFile发送AMOP消息文件
# 话题名：testTopic，是否广播：false(即使用单播)，文件：dist/conf/ca.crt， 发送次数：1次
java -cp "apps/*:lib/*:conf/" org.fisco.bcos.sdk.demo.amop.tool.AmopPublisherFile testTopic false ../docs/images/FISCO_BCOS_Logo.svg 1
```

终端输出

```bash
3s ...
2s ...
1s ...
start test
===================================================================
Step 1: Send out msg, topic:testTopic content: file ../docs/images/FISCO_BCOS_Logo.svg
Step 3:Get response, { errorCode:0 error:null seq:6e6a1e23d7ca47a0a1904bcb0a151f51 content:Yes, I received! }
```

订阅者终端输出

```bash
Start test
Step 2:Receive file, filename length:34 filename binary:[46, 46, 47, 100, 111, 99, 115, 47, 105, 109, 97, 103, 101, 115, 47, 70, 73, 83, 67, 79, 95, 66, 67, 79, 83, 95, 76, 111, 103, 111, 46, 115, 118, 103] filename:../docs/images/FISCO_BCOS_Logo.svg
|---save file:../docs/images/FISCO_BCOS_Logo.svg success
|---response:Yes, I received!
```

#### 私有话题Demo

**运行订阅者:**

```bash
# 话题名：PrivateTopic，私钥文件：conf/amop/consumer_private_key.p12， 文件密码：123456
java -cp "apps/*:lib/*:conf/" org.fisco.bcos.sdk.demo.amop.tool.AmopSubscriberPrivate PrivateTopic conf/amop/consumer_private_key.p12 123456
```

终端输出

```
Start test
```

**单播私有话题消息**：

```bash
# 话题名：PrivateTopic，公钥1:conf/amop/consumer_public_key_1.pem，公钥2:null， 是否广播：false(即使用单播)，内容：Tell you something， 发送次数：2次
java -cp "apps/*:lib/*:conf/" org.fisco.bcos.sdk.demo.amop.tool.AmopPublisherPrivate PrivateTopic conf/amop/consumer_public_key_1.pem null false "Tell you something" 2
```

**广播消息**：

```bash
# 话题名：PrivateTopic，公钥1:conf/amop/consumer_public_key_1.pem，公钥2:null， 是否广播：true，内容：Tell you something， 发送次数：1次
java -cp "apps/*:lib/*:conf/" org.fisco.bcos.sdk.demo.amop.tool.AmopPublisherPrivate PrivateTopic conf/amop/consumer_public_key_1.pem null true "Tell you something" 1
```

**发送文件**：

```bash
# 调用AmopPublisherPrivateFile发送AMOP消息文件
# 话题名：PrivateTopic，公钥1:conf/amop/consumer_public_key_1.pem，公钥2:null， 是否广播：false(即使用单播)，文件：../docs/images/FISCO_BCOS_Logo.svg， 发送次数：1次
java -cp "apps/*:lib/*:conf/" org.fisco.bcos.sdk.demo.amop.tool.AmopPublisherPrivateFile PrivateTopic conf/amop/consumer_public_key_1.pem null false ../docs/images/FISCO_BCOS_Logo.svg 1
```



## 5. 错误码

- 99：发送消息失败，AMOP经由所有链路的尝试后，消息未能发到服务端，建议使用发送时生成的`seq`，检查链路上各个节点的处理情况。
- 100：区块链节点之间经由所有链路的尝试后，消息未能发送到可以接收该消息的节点，和错误码‘99’一样，建议使用发送时生成的‘seq’，检查链路上各个节点的处理情况。
- 101：区块链节点往Sdk推送消息，经由所有链路的尝试后，未能到达Sdk端，和错误码‘99’一样，建议使用发送时生成的‘seq’，检查链路上各个节点以及Sdk的处理情况。
- 102：消息超时，建议检查服务端是否正确处理了消息，带宽是否足够。
- 103：因节点出带宽限制，SDK发到节点的AMOP请求被拒绝。