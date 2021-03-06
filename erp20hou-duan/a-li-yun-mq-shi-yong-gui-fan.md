# 阿里云MQ使用规范

### 使用环境:

```
springboot 2.0版本以上
jdk1.8+
```

### 使用maven 的jar包

```
<dependency>
    <groupId>cn.lbonline</groupld>
    <artifactId>lbmq-spring-boot-starter</artifactId>
    <version>1.0.0.RELEASE</version>
</dependency>
```

### application.yml配置

##### 1.无序消息

```
  aliyun: 
    mq:
      onsAddr: 联系运维夏康
      accessKey: 联系运维夏康
      secretKey: 联系运维夏康
      producer:
        enabled: true/false
        producerId: 联系运维夏康
      consumer:
        enabled: true/false
        consumerId: 联系运维夏康
      sendMsgTimeoutMillis: 3000 //可选,发送超时时间，单位毫秒
      suspendTimeMillis: 100 //可选,重试前的等待时间，单位毫秒
      maxReconsumeTimes: 20 //可选,最大重试次数
```

##### 2.有序消息

```
aliyun: 
    mq:
      onsAddr: 联系运维夏康
      accessKey: 联系运维夏康
      secretKey: 联系运维夏康
      order-producer:
        enabled: true/false
        producerId: 联系运维夏康
      order-consumer:
        enabled: true/false
        consumerId: 联系运维夏康
      sendMsgTimeoutMillis: 3000 //可选,发送超时时间，单位毫秒
      suspendTimeMillis: 100 //可选,重试前的等待时间，单位毫秒
      maxReconsumeTimes: 20 //可选,最大重试次数
```

### DEMO演示

##### 1.无序消息

* 发送消息

```
public class SendService {

    @Value("${topic}")
    private String topic;

    @Value("${tag}")
    private String tag;

    @Autowired 
    private ProducerService producerService; //无序消息生成者

    public ResultVo sendMessage() {
        // 封装消息
        MessageEvent messageEvent = new MessageEvent();

        // 消息体 
        messageEvent.setMessageBody(new Object());
        // 主题
        messageEvent.setTopic(topic);
        // 标示
        messageEvent.setTag(tag);
        // 业务唯一key
        messageEvent.setMessageKey(messageEvent.generateMessageKey());
        // 发送消息
        producerService.disorderSend(messageEvent);

        return ResultVoUtil.success();
    }
}
```

* ##### 消费消息 

```
@Component
@LbMQMessageListener(topic = "需要监听的主题", tag = "标示") //tag为过滤标示,支持 * 、tagA||tagB等格式
public class MessageListener extends AbstractMessageListener<MessageEvent> {

    @Override
    public void handle(MessageEvent event) {
        System.out.println(event.getMessageBody());
    }
}
```

* ##### api

```
     /**
     * 发送无序消息(同步)
     * @param event
     * @return SendResult
     */
    public SendResult disorderSend(MessageEvent event);

    /**
     * 发送延时消息(同步)
     * @param event
     * @param delayTime 延时时间(毫秒),最长40天
     * @return
     */
    public SendResult delaySend(MessageEvent event, long delayTime);

    /**
     * 发送定时消息(同步)
     * @param event
     * @param timeStamp 具体时间的时间戳
     * @return
     */
    public SendResult timingSend(MessageEvent event, long timeStamp);

    /**
     * 获取同步发送消息结果
     * @param message
     * @return
     */
    private SendResult getSynchSendResult(Message message);

    /**
     * 发送无序消息(异步)
     * @param event
     */
    public void asyncSend(MessageEvent event);

    /**
     * 发送延时消息(异步)
     * @param event
     * @param delayTime
     */
    public void asyncDelaySend(MessageEvent event, long delayTime);

    /**
     * 发送定时消息(异步)
     * @param event
     * @param timeStamp
     */
    public void asyncTimingSend(MessageEvent event, long timeStamp);

    /**
     * 发送单向消息
     * @param event
     */
    public void oneWaySend(MessageEvent event);
```

##### 2.顺序消息

* 发送消息

```
public class SendService {

    @Value("${topic}")
    private String topic;

    @Value("${tag}")
    private String tag;

    @Autowired 
    private OrderProducerService orderProducerService; //顺序消息生成者

    public ResultVo sendMessage() {
        // 封装消息
        MessageEvent messageEvent = new MessageEvent();

        // 消息体 
        messageEvent.setMessageBody(new Object());
        // 主题
        messageEvent.setTopic(topic);
        // 标示
        messageEvent.setTag(tag);
        // 业务唯一key
        messageEvent.setMessageKey(messageEvent.generateMessageKey());
        // 发送消息
        orderProducerService.orderSend(messageEvent);

        return ResultVoUtil.success();
    }
}
```

* ##### 消费消息

```
@Component
@LbMQMessageListener(topic = "需要监听的主题", tag = "标示") //tag为过滤标示,支持 * 、tagA||tagB等格式
public class MessageListener extends AbstractOrderMessageListener<MessageEvent> {

    @Override
    public void handle(MessageEvent event) {
        System.out.println(event.getMessageBody());
    }
}
```

* ##### api

```
    /**
     * 发送顺序消息(默认全局分区,同步)
     * @param event
     * @return SendResult
     */
    public SendResult orderSend(MessageEvent event) {
        log.info("Synch order send message topic:{}, tag:{}", event.getTopic(), event.getTag());

        if (StringUtils.isEmpty(event.getTopic()) || null == event.getMessageBody()) {
            log.error("Topic or body is null");

            throw new ONSClientException("Topic or body is null");
        }

        Message message = new Message(event.getTopic(), event.getTag(), SerializationUtils.serialize(event));

        if (StringUtils.isEmpty(event.getShardingKey())) {
            event.setShardingKey(DEFULT_SHARDING_KEY);
        }

        if (StringUtils.isEmpty(event.getMessageKey())) {
            String key = event.generateMessageKey();

            log.info("消息业务属性key:{}", key);

            // 设置代表消息的业务关键属性，请尽可能全局唯一
            // 以方便您在无法正常收到消息情况下，可通过 MQ 控制台查询消息并补发。
            // 注意：不设置也不会影响消息正常收发
            message.setKey(key);
        }

        // 分区顺序消息中区分不同分区的关键字段，sharding key 于普通消息的 key 是完全不同的概念。
        // 全局顺序消息，该字段可以设置为任意非空字符串。
        String shardingKey = event.getShardingKey();

        SendResult result = null;
        try {
            result = orderProducer.send(message, shardingKey);

            log.info("Send message success result:{}", result.toString());
        } catch (Exception e) {
            log.error("Send message fail topic:{}, reason:{}", result.getTopic(), e.getMessage());

            e.printStackTrace();
        }

        return result;
    }
```

##### 3 .参考

```
https://help.aliyun.com/product/29530.html?spm=a2c4g.11186623.6.539.92415fa6aFPwu0
https://github.com/jibaole/spring-boot-starter-alimq
```



