# activemq

### 作用
1. 解耦 
2. 削峰
3. 异步

### 队列与主题（topic）
- 队列---发消息（一个接收者）
- 主题---对订阅的系统统一发送消息（多个接收者）

### 普通启动
1. 进入bin目录
2. 启动activemq     ./activemq start
3. activemq 默认端口是61616
4. 启动失败（hostname有下划线）
    ```
    vim /etc/hostname
    
    reboot
    ```
5. 查看启动
    ```
    ps -ef|grep activemq
    ps -ef|grep activemq|grep -v grep    // 过滤掉grep
    netstat -anp|grep 61616
    lsof -i:61616
    ```
6. 重启
    ```
    ./activemq restart
    ```
7. 关闭
    ```
    ./activemq stop
    ```
### 带运行日志的方式启动
    ```
    ./bin/activemq start > data/log.log
    ```

### 查看控制台
- ip:8161
- admin/admin
- 61616端口提供JMS服务
- 8161端口提供管理控制台服务

### 简单发布==队列==到MQ
  ```
  private static final String ACTIVE_MQ_URL = "tcp://49.235.209.207:61616";
    private static final String QUEUE_NAME = "queue01";

    public static void main(String[] args) throws JMSException {
        // 1. 创建连接工厂 这个构造方法采用默认用户名密码
        ActiveMQConnectionFactory activeMQConnectionFactory = new ActiveMQConnectionFactory(ACTIVE_MQ_URL);
        // 2. 获得connection对象 并启动访问
        Connection connection = activeMQConnectionFactory.createConnection();
        connection.start();

        // 3. 创建会话session
        // 两个参数，第一个叫事务，第二个叫签收
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);

        // 4. 创建目的地（队列/主题）
        Queue queue = session.createQueue(QUEUE_NAME);
        //        Destination destination = session.createQueue(QUEUE_NAME);
        // 5. 现在创建消息的生产者
        MessageProducer messageProducer = session.createProducer(queue);
        // 6. 使用messageProducer创建3条消息，发送到mq
        for (int i=1; i<=3; i++){
            // 7.创建消息
            TextMessage textMessage = session.createTextMessage("msg---" + i);// 理解为一个字符串
            // 8.通过消息生产者发布messageProducer
            messageProducer.send(textMessage);
        }
        // 9.释放资源（关闭连接）
        messageProducer.close();
        session.close();
        connection.close();

        System.out.println("****消息发布到MQ成功!!");
    }
  ```
  
### 简单接收MQ队列
1. 同步阻塞方式(receive())-订阅者或者接收者调用==MessageConsumer的receive()方法==，receive方法在能够接收消息之前(或超时之前)将一直阻塞
接收到的massage要与发送的类型一致
```
while (true){
    TextMessage textMessage = (TextMessage) messageConsumer.receive();
    if(null != textMessage){
        System.out.println("接收到的消息：" + textMessage.getText());
    }else {
        break;
    }
}

messageConsumer.close();
session.close();
connection.close();
```
2. ==通过messageConsumer== ==的setMessageListener== 注册一个监听器，当有消息发送来时，系统自动调用MessageListener 的 ==onMessage== 方法处理消息.==异步==
```
     messageConsumer.setMessageListener(new MessageListener() {   // 可以用监听器替换之前的同步receive 方法
            public void onMessage(Message message)  {
                if (null != message  && message instanceof TextMessage){
                    TextMessage textMessage = (TextMessage)message;
                    try {
                        System.out.println("MessageListener****消费者的消息："+textMessage.getText());
                    }catch (JMSException e) {
                        e.printStackTrace();
                    }
                }
            }
        });
//        System.in.read();

        messageConsumer.close();
        session.close();
        connection.close();
```
### 多个消费者
1. 先生产，启动A消费者。问A消费者能消费到吗?
可以
2. 先生产，先启动A消费者，在启动B消费者。问B消费者能消费到吗？
A可以消费,B消费者不能被消费
3.先启动两个消费者，在生产2套消息，情况如何？
平分


## topic
1. 生产者将消息发布到topic中，==每个消息可以有多个消费者==。属于1：N的关系
2. 生产者和消费者之间有时间上的相关性，订阅某一主题的消费者==只能消费其订阅之后发布的消息==。
3. 无人订阅去生产消息，那就是一条废消息。一般==先启动消费者，在启动生产者==

代码的改动
```
Topic topic = session.createTopic(TOPIC_NAME);
```


比较项目 | topic队列模式 | Queue模式队列
---|--- | ---
工作模式 | "订阅-发布"模式，如果没有消费者，消息会被丢弃。如果有多个订阅者，这些订阅者都会收到消息。 | "负载均衡"模式，如果当前没有消费者，消息不会丢弃。如果有多个消费者，一条消息也只能给其中一个消费者。
有无状态 | 无状态 | Queue数据默认会在mq服务器上以文件形式保存，比如activemq一般会保存在$AMQ_HOME\data\kr-store\data下面，也可以配置成DB储存
传递完整性 | 如果没有消费者消息将会被丢弃 | 消息不会被丢弃
处理效率 | 由于要按照订阅者数量进行复制，所以处理性能会随着订阅和数量增加而明显降低，并且还要结合不同消息协议自身的性能差异 | 由于一条消息只发送给一个消费者，所以就算消费者再多，性能也不会有明显降低。当然不同消息协议的拘役性能也是有差异的。

### JMS结构和特点
- JMS provider : 实现JMS接口和规范的中间件，即MQ服务器
- JMS producer : 创建和发送jms消息的客户端应用
- JMS consumer : 接收和处理jms消息的客户端应用
- JMS message  ：消息头，消息属性，消息体

### JMS message
#### 1. 消息头
- JMSDestination:消息发送的目的地，主要指queue和topic
- JMSDeliveryMode:持久模式和非持久模式。  
  ==一条持久行的消息==：应该被传送"一次仅仅一次"，这就意味着JMS服务器故障时，重新恢复后，此消息可以再次传  
  ==一条非持久的消息==：最多传递一次.服务器故障时，该消息永久丢失
- JMSExpiration
  可以设置消息在一定时间后过期，默认是永不过期
- JMSPriority
  消息优先级，从0-9是个级别，0-4是普通消息，5-9是加急消息。  
  JMS不要去MQ严格按照这10个优先级发送消息，但必须保证加急高于普通。==默认是4==
- JMSMessageId
  唯一识别每个消息的标识，有MQ产生
#### 2.消息体
- 封装具体的消息数据
- 5种消息体格式  
    ==TextMessage==:一个普通字符串消息，包含一个String,  
    ==MapMessage==：一个Map类型的消息，key为String，而值为java基本类型,  
BytesMessage：二进制数组消息，包含一个byte[],   
StreamMessage：java数据流消息，用标准流操作来顺序的填充和读取,  
ObjectMessage：对象消息，包含一个可序列化的java对象
- 发送和接收的消息体类型必须一致
#### 3.消息属性
- 如果需要出消息头字段意外的值，那么可以使用消息属性
- 识别/去重/重点标注等操作非常有用的方法   


### JMS消息的可靠性
1. presistent:持久性(队列默认是持久化的)
- 参数设置  
  非持久化，服务器宕机消息丢失
  ```
  messageProducer.setDeliveryMode(DeliveryMode.NON_PERSISTENT);
  ```
  持久化，服务器宕机消息依然存在
  ```
  messageProducer.setDeliveryMode(DeliveryMode.PERSISTENT);
  ```
  关于topic的持久性:  
  持久化的topic，只要是在==持久化的订阅者==之后发布的消息，持久化订阅者都会受到
  ①.topic持久性需要生产者将消息变成持久化  
  需要创建将topic消息设置为持久化的,连接启动放在后面

```
    // 1. 创建连接工厂 这个构造方法采用默认用户名密码
        ActiveMQConnectionFactory activeMQConnectionFactory = new ActiveMQConnectionFactory(ACTIVE_MQ_URL);
        // 2. 获得connection对象 并启动访问
        Connection connection = activeMQConnectionFactory.createConnection();
        // 3. 创建会话session
        // 两个参数，第一个叫事务，第二个叫签收
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        // 4. 创建目的地（队列/主题）
        Topic topic = session.createTopic(TOPIC_NAME);
//        Destination destination = session.createQueue(QUEUE_NAME);
        // 5. 现在创建消息的生产者
        MessageProducer messageProducer = session.createProducer(topic);
        messageProducer.setDeliveryMode(DeliveryMode.PERSISTENT);

        connection.start();
        // 6. 使用messageProducer创建3条消息，发送到mq
        for (int i=1; i<=3; i++){
            // 7.创建消息
            TextMessage textMessage = session.createTextMessage("msg---" + i);// 理解为一个字符串
            // 8.通过消息生产者发布messageProducer
            messageProducer.send(textMessage);
        }
        // 9.释放资源（关闭连接）
        messageProducer.close();
        session.close();
        connection.close();

        System.out.println("****topic01消息发布到MQ成功!!");
```
  ② 修改消费者  
    消费者添加clientId，
    创建持久化的消费者，然后启动

```
    System.out.println("..........zhang3");
        // 1. 创建连接工厂 这个构造方法采用默认用户名密码
        ActiveMQConnectionFactory activeMQConnectionFactory = new ActiveMQConnectionFactory(ACTIVE_MQ_URL);
        // 2. 获得connection对象 并启动访问
        Connection connection = activeMQConnectionFactory.createConnection();
        // 创建clientid
        connection.setClientID("zhang3");
        // 3. 创建会话session
        // 两个参数，第一个叫事务，第二个叫签收
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        // 4. 创建目的地（队列/主题）
        Topic topic = session.createTopic(TOPIC_NAME);
        TopicSubscriber topicSubscriber = session.createDurableSubscriber(topic, "remake......");
        connection.start();

        Message message = topicSubscriber.receive();
        while (null != message) {
            TextMessage textMessage = (TextMessage) message;
            System.out.println("接收到的持久化topic消息：" + textMessage.getText());
        }
        session.close();
        connection.close();
```
  
2. 事务（偏提供者）
- false: 只需要调用send方法，消息就会进入队列(可以在控制台看见)  
  关闭事务，那么第二个参数需要设置成有效
- true:  先执行send，在执行commit，消息才能真正的提交到队列中。  
  消息需要批量发送，则需缓冲区处理  
使用session提交事务
```
    try{
            // 业务代码
            session.commit();
        }catch (Exception e){
            session.rollback();
            e.printStackTrace();
        }finally {
            if(null != session){
                session.close();
            }
        }
```
- 消费者将事务改为true，消费者消费完之后也需要提交事务，可以避免重复消费。
3. Ackonwledge:签收（默认是自动）
- 没有事务：当签收设置为Session.CLIENT_ACKNOWLEDGE，需要对每一条消费的消息进行反馈textMessage.acknowledge();
- 有实物： 当消费者有实物，将签收设置为手动签收，只需要提交事务即可

### Broker(相当于activemq服务器的一个实例)
Broker其实就是实现了用代码的形式启动ActiveMq，将MQ迁入到java代码中，以便于随时启动。
### 通过不同配置文件启动
```
./bin/activemq start xbean:file:/usr/local/apache-activemq-5.15.9/conf/activemq2.xml
```

### 创建本地broker
1. pom中引入
```
<!-- 创建本地borker 解决报错 -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.9.8</version>
</dependency>
```
2. 代码示例
```
// ActiveMq也支持在vm中基于嵌入式的broker
BrokerService brokerService = new BrokerService();
brokerService.setUseJmx(true);
brokerService.addConnector("tcp://localhost:61616");
brokerService.start();
```
相当于本地启动了一个微型mq服务。连接地址：tcp:localhost:61616

### spring整合activeMQ
#### 队列
1. pom修改
```
<!--  activeMQ  jms 的支持  -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jms</artifactId>
    <version>4.3.23.RELEASE</version>
</dependency>
<dependency>    <!--  pool 池化包相关的支持  -->
    <groupId>org.apache.activemq</groupId>
    <artifactId>activemq-pool</artifactId>
    <version>5.15.9</version>
</dependency>


<dependency>
    <groupId>org.apache.activemq</groupId>
    <artifactId>activemq-all</artifactId>
    <version>5.15.9</version>
</dependency>
        <dependency>
    <groupId>org.apache.xbean</groupId>
    <artifactId>xbean-spring</artifactId>
    <version>3.16</version>
</dependency>
```
2. applicationContext配置
```
    <!-- 开启自动扫描的包 -->
    <context:component-scan base-package="com.mq.demo"/>

    <!-- 配置生产者 -->
    <bean id="jmsFactory" class="org.apache.activemq.pool.PooledConnectionFactory" destroy-method="stop">
        <property name="connectionFactory">
            <!-- 真正可以提供connection的connectionFactory，有对应的jms服务厂商提供的-->
            <bean class="org.apache.activemq.ActiveMQConnectionFactory">
                <property name="brokerURL" value="tcp:49.235.209.207:61616"/>
            </bean>
        </property>
        <property name="maxConnections" value="100"></property>
    </bean>

    <!-- 这个是队列目的地，点对点 (通过构造器注入，来添加queue)-->
    <bean id="destinationQueue" class="org.apache.activemq.command.ActiveMQQueue">
        <constructor-arg index="0" value="spring-active-queue"/>
    </bean>

    <!-- spring提供的jms工具类，可以进行消息发送、接收等 -->
    <bean id="jmsTemplate" class="org.springframework.jms.core.JmsTemplate">
        <property name="connectionFactory" ref="jmsFactory"/>
        <property name="defaultDestination" ref="destinationQueue"/>
        <property name="messageConverter">
            <bean class="org.springframework.jms.support.converter.SimpleMessageConverter"/>
        </property>
    </bean>
```
3. 代码  
其目的地都是在配置文件中注入了  
生产者
```
ApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml");

        SpringProducer springProducer = (SpringProducer) ctx.getBean("springProducer");

        springProducer.jmsTemplate.send(new MessageCreator() {
            @Override
            public Message createMessage(Session session) throws JMSException {
                TextMessage textMessage = session.createTextMessage("****spring和activeMq整合case");
                return textMessage;
            }
        });
        System.out.println("producer----发送成功!!!");
```
消费者
```
        ApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml");
        SpringConsumer springConsumer = (SpringConsumer) ctx.getBean("springConsumer");
        String value = (String) springConsumer.jmsTemplate.receiveAndConvert();
        System.out.println("****spring和activeMq整合case" + value);
```
#### topic-主题
可以只修改配置文件，代码还是queue代码，但是要注意启动顺序
```
    <!-- 这个是队列 -->
    <bean id="destinationTopic" class="org.apache.activemq.command.ActiveMQTopic">
        <constructor-arg index="0" value="spring-active-topic"/>
    </bean>

    <!-- spring提供的jms工具类，可以进行消息发送、接收等 -->
    <bean id="jmsTemplate" class="org.springframework.jms.core.JmsTemplate">
        <property name="connectionFactory" ref="jmsFactory"/>
        <property name="defaultDestination" ref="destinationTopic"/>
        <property name="messageConverter">
            <bean class="org.springframework.jms.support.converter.SimpleMessageConverter"/>
        </property>
    </bean>
```
#### 在spring中实现消费者不启动，直接通过配置监听完成
```
    <bean id="jmsContainer" class="org.springframework.jms.listener.DefaultMessageListenerContainer">
        <property name="connectionFactory" ref="jmsFactory"/>
        <property name="destination" ref="destinationTopic"/>
        <!-- public class MyMessageListener implements MessageListener -->
        <property name="messageListener" ref="myMessageListener"/>
    </bean>
```
代码
```
@Component
public class MyMessageListener implements MessageListener {
    @Override
    public void onMessage(Message message) {
        if(null != message && message instanceof TextMessage){
            TextMessage textMessage = (TextMessage) message;
            try {
                System.out.println("监听器收到的消息：" + ((TextMessage) message).getText());
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }
}
```

### SpringBoot整合ActiveMq
@EnableJms 用来声明对 JMS 注解的支持
#### queue
1.修改pom
```
<!-- mq -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-activemq</artifactId>
    <version>2.1.5.RELEASE</version>
</dependency>
```
2. 配置配置文件
```
server:
  port: 7777

spring:
  activemq:
    broker-url: tcp://49.235.209.207:61616  # MQ服务器地址
    user: admin
    password: admin
  jms:
    pub-sub-domain: false  #false = queue，true=Topic


#自己定义名称
myqueue: boot-activemq-queue
```
3. 添加配置类
```
@Component
@EnableJms
public class ConfigBean {

    @Value("${myqueue}")
    private String myQueue;

    @Bean
    public Queue queue(){
        return new ActiveMQQueue(myQueue);
    }
}
```
4. 提供者代码
```
@Component
public class QueueProducer {

    @Autowired
    private JmsMessagingTemplate jmsMessagingTemplate;

    @Autowired
    private Queue queue;

    public void produceMsg(){
        jmsMessagingTemplate.convertAndSend(queue, "********"+ UUID.randomUUID().toString());
    }
}
```
5. 定时发送(在主启动类开启@EnableScheduling注解)
```
    @Scheduled(fixedDelay = 3000)
    public void produceMsgScheduled(){
        jmsMessagingTemplate.convertAndSend(queue, "********"+ UUID.randomUUID().toString());
        System.out.println("发送了一条消息;");
    }
```
6. 消费者代码
```
@Component
public class QueueComsumer {

    @JmsListener(destination = "${myqueue}")
    public void receive(TextMessage textMessage)throws JMSException{
        System.out.println("*************消费者收到消息" + textMessage.getText());
    }
}
```

#### topic
##### 提供者
1.配置文件
pub-sub-domain设置为true
```
jms:
    pub-sub-domain: true  #false = queue，true=Topic
```
2. 配置bean
```
@Component
public class ConfigBean {

    @Value("${mytopic}")
    private String topicName;

    @Bean
    public Topic topic(){
        return new ActiveMQTopic(topicName);
    }
}
```
3. 代码
```
@Autowired
    private JmsMessagingTemplate jmsMessagingTemplate;
    @Autowired
    private Topic topic;

    public void produceTopic(){
        jmsMessagingTemplate.convertAndSend(topic, "主题消息:" + UUID.randomUUID().toString());
        System.out.println("发布topic成功");
    }
```
##### 订阅者
```
@Component
public class TopicConsumer {

    @JmsListener(destination = "${mytopic}")
    public void receive(TextMessage textMessage){
        try {
            System.out.println("订阅者订阅到的主题消息：" + textMessage.getText());
        } catch (JMSException e) {
            e.printStackTrace();
        }
    }
}
```

### 协议
- ActiveMQ直接的client-broker通讯协议有:TCP、NIO、UDP、SSL、Http(s)、VM 
- 其中配置Transport Connector的文件在activeMq安装目录的conf/activemq.xml中的<transportConnectors>标签内，默认就是openwire-tcp
```
        <transportConnectors>
            <!-- DOS protection, limit concurrent connections to 1000 and frame size to 100MB -->
            <transportConnector name="openwire" uri="tcp://0.0.0.0:61616?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
            <transportConnector name="amqp" uri="amqp://0.0.0.0:5672?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
            <transportConnector name="stomp" uri="stomp://0.0.0.0:61613?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
            <transportConnector name="mqtt" uri="mqtt://0.0.0.0:1883?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
            <transportConnector name="ws" uri="ws://0.0.0.0:61614?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
        </transportConnectors>
```
**- tcp**
1. 这个是Borker默认配置，TCP的Client监听端口是61616
2. 在网络传输数据前，必须要序列化数据，消息是同脚一个叫wire proctocl的来序列化字节流。默认情况下ActiveMQ把wire protocol叫做OpenWire，他的目的是促使网络上的效率和数据快速交互
3. TCP连接的URL形式如：tcp:hostname:port?key=valeu&key=value 后面的参数是可选的
4. TPC传输的优点: 
   1.   TCP协议传输可靠性高，稳定性强
   2.   高效性：字节流方式传递，效率很高    
4.3 有效性、可用性：应用广泛，支持任何平台
5. 官网参考:http://activemq.apache.org/tcp-transport-reference

**- NIO**
1. NIO协议和TCP协议类似，但是NIO更侧重于底层的访问操作。他允许开发人员对统一资源可有更多的client调用和服务端有更多的负载。
2. 适合使用NIO协议的场景  
   1.   可能有大量的cient
去连接到Broker上。一般情况下，大量的Client去连接是被操作系统的线程所限制的。一次NIO的实现比TCP需要更少的线程去运行，所以建议使用NIO协议。  
   2.  可能对于Broker有一个很迟钝的网络传输，NIO比TCP提供更好的性能
3. NIO连接URL形式： nio://hostname:port?key=value
4. 官网参考:http://activemq.apache.org/nio-transport-reference
5. 配置NIO
   1.   修改activmq.xml配置文件（修改之前备份）。如果仅仅添加nio协议，并不会影响之前tcp
   ```
       <transportConnector name="nio" uri="nio://0.0.0.0:61616?trace=true"/>  
    ```
   2.   代码修改(普通只需换个url)
   ```
       private static final String ACTIVE_MQ_URL = "nio://49.235.209.207:61618";
   ```
   3. AUTO NIO 想要一个端口既支持NIO，也支持其他传输协议。http://activemq.apache.org/auto
   4. 修改
   ```  
    <transportConnector name="auto+nio" uri="auto+nio://0.0.0.0:61608?maxumumConnections=1000&amp;wireFormat.maxFrameSize=104857600&amp;org.apache.activemq.transport.nio.SelectorManager.corePoolSize=20&amp;org.apache.activemq.transport.nio.SelectorManager.maximumPoolSize=50"/>
   ```
   使用两种（多种）连接都可以
   ```
   "tcp://49.235.209.207:61608";
   "nio://49.235.209.207:61608";
   ```
 
 ### ActiveMQ的持久化
1. MQ服务器宕机了，消息不会丢失(最佳的保存方式是物理保存，不要和mq在同一台服务器上)。ActiveMQ持久化机制有：JDBC、AMQ、KahaDB、levelDB等。   
2. 消息发送出去后，mq将消息存储到数据库等，成功发送则删除，时报则重新发送。
3. 消息中心启动成功后，先检查指定的存储位置，如有消息未发送出去，则先发送。

#### AMQ message store
AMQ是一种文件存储形式，具有写入快和容易恢复的特点。消息存储在一个个文件中，文件默认大小为32M，当一个存储文件的消息已经被全部消费，那么这个文件将被表示为可删除，在下一个清除阶段，这个文件被删除。==AMQ使用5.3之前的版本==

#### KahaDB消息存储(默认)
- 基于日志文件，从activeMQ5.4开始默认的持久化插件(ActiveMQ.xml)
```
 <persistenceAdapter>
            <kahaDB directory="${activemq.data}/kahadb"/>
</persistenceAdapter>

```
- 消息存储使用一个==事务日志==和仅仅是要用一个==索引文件==来存储它所有的地址
- 数据被追加到data logs中。当不在需要logs是，将会被丢弃
- db<Number>.log 存储。默认 32M一个文件，当不在有引用到数据文件的任何消息时，文件会被删除或者归档。
- db.data 该文件包含了持久化的Btree所有。指向log的数据。
- db.free 当前db.data那些页面是空的，文件具体内容是所有空闲页的ID
- db.redo 用来进行消息回复，如果KahaDB消息存储在强制退出后启动，用于回复BTree索引
- lock文件锁，标识当前获得kahadb读写权限的broker

#### LevelDB
- ActiveMQ 是5.8以后之后引入的，与KahaDB非常相似，也是基于文件的本地数据库储存形式，比KahaDB更快的持久性
- 配置
```
 <persistenceAdapter>
            <LevelaDB directory="${activemq.data}/kahadb"/>
</persistenceAdapter>
```
#### JDBC消息存储(msql)
1. 添加mysql驱动到lib
2. jdbcPersistenceAdapter配置
```
<persistenceAdapter>
	<jdbcPersistenceAdapter dataSource="#mysql-ds" createTablesOnStartup="true"/>
</persistenceAdapter>
```
dataSource指定将要引用的持久化数据库的bean名称，createTableOnStartup是否在启动的时候创建数据表，默认是true，这样每次启动都会创建数据表，==一般是第一次启动的时候设为true之后改为false。==
3. 连接池配置(放在borker标签之外)
```
<bean id="mysql-ds" class="org.apache.commons.dbcp2.BasicDataSource" destroy-method="close">
	<property name="driverClassName" value="com.mysql.jdbc.Driver"/>
	<property name="url" value="jdbc:mysql://cdb-fj96zrqa.cd.tencentcdb.com:10062/activemq?relaxAutoCommit=true"/>
	<property name="username" value="root"/>
	<property name="password" value="root!@#$"/>
	<property name="maxTotal" value="200"/>
	<property name="poolPreparedStatements" value="true"/>
</bean>
```
4. 建仓SQL和建表说明  
- activemq_acks：用于存储订阅关系。如果是持久化Topic，订阅者和服务器的订阅关系在这个表保存
- activemq_lock：在集群环境中才有用，只有一个Broker可以获得消息，称为Master Broker
- activemq_msgs：用于存储消息，Queue和Topic都存储在这个表中
5. 验证(一定要开启生产者的持久化)
```
producer.setDeliveryMode(DeliveryMode.PERSISTENT);
```
- 生产者（DeliveryMode.NON_PERSISTENT）：消息保存到内存中
- 生产者（DeliveryMode.PERSISTENT）：消息保存到持久化(保存到形影的文件或者数据库)
#### JDBC Message store ActiveMQ Journal(高速缓存,即在mysql与mq中间加一层)
- 这种方式克服了JDBC Store的不足，JDBC每次消息过来，都需要去写库和读库。
- ActiveMQ Journal,==使用高速缓存写入技术，大大提高了性能。==
- 当消费者的消费速度能够及时跟上生产者消息的生产速度时，journal文件能够大大减少需要写入到DB中的消息。
- 如果消费者的消费速度很慢，这个时候journal文件可以使消息以批量方式写到DB。
- 举个例子：  
 生产者生产了1000条消息，这1000条消息会保存到journal文件，如果消费者的消费速度很快的情况下，在journal文 件还没有同步到DB之前，消费者已经消费了90%的以上的消息，那么这个时候只需要同步剩余的10%的消息到DB。如果消费者的消费速度很慢，这个时候journal文件可以使消息以批量方式写到DB。
1. 配置
```
<persistenceFactory>
     <journalPersistenceAdapterFactory 
     	journalLogFiles="4"
		journalLogFileSize="32768"
		useJournal="true"
		useQuickJournal="true"
		dataSource="#mysql-ds"
		dataDirectory="activemq-data"/>
</persistenceFactory>
```


### 集群

主从类型 | 要求 | 优点 | 缺点
---|--- | ---|--- 
共享文件系统主从 | 共享文件系统，例如SAN | 	根据需要运行尽可能多的从站。Automatic recovery of old masters | 需要共享文件系统
JDBC Master Slave | 共享数据库 | 根据需要运行尽可能多的从站。Automatic recovery of old masters | 需要共享数据库。同样相对较慢，因为它不能使用高性能期刊
复制的LevelDB存储 | ZooKeeper服务器 | 根据需要运行尽可能多的从站。Automatic recovery of old masters。==非常快。== | 需要ZooKeeper服务器。 


#### Zookeeper和Replicated LevelDB集群原理
引入消息队列之后该如何保证其高可用性 --> 集群   
- 基于==ZooKeeper和LevelDB==搭建ActiveMQ集群。
- 集群仅提供==主备方式==的高可用集群功能，避免单点故障。


#### zookeeper+replicated-leveldb-store的主从集群
##### 原理:
1. 使用ZooKeeper集群注册所有的ActiveMQ Broker但只有其中的一个Broker可提供服务它将被视为Master, ==其他的Broker处于待机状态被视为Slave。==
2. 如果Master因故障而不能提供服务ZooKeeper会从Slave中选举出一- 个Broker充当Master。
3. Slave连接Master并同步他们的存储状态，slave 不接受客户端连接。所有的存储操作都将被复制到连接至Master的Slaves。如果Master宕机得到了最新更新的Slave会成为Master。故障节点在恢复后会重新加入到集群中并连接Master进入Slave 模式。
4. 所有需要同步的消息操作都将等待存储状态被复制到其他法定节点的操作完成才能完成。
```
所以，如果你配置了replicas=3, 那么法定大小是(3/2)+1=2。
Master 将会存储并更新然后等待(2-1)=1 个Slave 存储和更新完成,才汇报success。
有一个node要作为观擦者存在。
当一个新的Master被选中，你需要至少保障--个法定node在线以能够找到拥有最新状态的node。这个node才可以成为新的Master。
```
### 集群
1. mq规划

主机 | zookeeper集群端口 | AMQ集群bind端口 | AMQ消息tcp端口 |管理控制台端口 | AMQ节点安装目录
---| --- |--- |--- |--- |---
192.168.111.136| 2191 |  bind="tcp://0.0.0.0:63631" | 61616 |  8161 | /mq_ cluster/mq_ node01
192.168.111.136| 2192 |  bind="tcp://0.0.0.0:63632" | 61618 |  8162 | /mq_ cluster/mq_ node02
192.168.111.136| 2193 |  bind="tcp://0.0.0.0:63633" | 61619 |  8163 | /mq_ cluster/mq_ node03

2. 创建3台集群目录(先创建3台mq服务器)
```
cp -r apache-activemq-5.15.9 mq_node01
cp -r mq_node01 mq_node02
cp -r mq_node01 mq_node03
```
3. 修改管理控制台端口
    1. mq_node01 默认不动
    2. 修改控制台端口(jeety.xml)
    ```
    109         <property name="host" value="0.0.0.0"/>
    110         <property name="port" value="8161"/>
    ```
    3. 修改mq端口
4. 修改ip映射
```
|[ rootozzyy etc]并 vim /etc/hosts
```
```
192.168.111.136 zzyymq- server
```
5. 3个节点的borkeName需要一致
6. 3个节点的持久化配置(不使用默认的db)
```
<persistenceAdapter>
    <replicatedLevelDB
      directory="${activemq.data}/leveldb"
      replicas="3"
      bind="tcp://0.0.0.0:61613"
   	  zkAddress="localhost:2191,localhost:2192,localhost:2193"
      hostname="192.168.199.23"
      sync="local_disk"
      zkPath="/activemq/leveldb-stores"
      />
</persistenceAdapter>
```
7. 修改各个节点的消息端口
```
<transportConnector name="openwire" uri="tcp://0.0.0.0:61616?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
```
8. 



### 其他
#### 引入消息队列之后该如何保证其高可用性
zookeeper+replicated-leveldb-store的主从集群

#### 异步投递Async Sends
1. 说明：对于一个slowConsumer,使用同步发送消息可能出现Producer堵塞等情况，慢消费者适合使用异步发送。
2. 这两种情况下是同步的：除非明确指定使用同步发送的方式或者在未使用事务的前提下发送持久化的消息
3. 如果你==没有使用事务且发送的是持久化的消息==，每--次发送都是同步发送的且会阻塞producer直到broker返回一个确认，表示消息已经被安全的持久化到磁盘。确认机制提供了消息安全的保障，但同时会阻塞客户端带来了很大的延时。
4. 很多高性能的应用，==允许在失败的情况下有少量的数据丢失==。如果你的应用满足这个特点，你可以使用异步发送来提高生产率，即使发送的是持久化的消息。
5. ==我们通常在发送消息量比较密集的情况下使用异步发送==，它可以很大的提升Producer性能;就是需要消耗较多的Client端内存同时也会导致broker端性能消耗增加;此外它不能有效的确保消息的发送成功。在useAsyncSend=true的情况下客户端需要容忍消息丢失的可能。   
解决方案：
- 1
```
cf = new ActiveMQConnectionFactory("tcp://locahost:61616?jms.useAsyncSend=true");
```
- 2
```
((ActiveMQConnectionFactory)connectionFactory).setUseAsyncSend(true);
```
- 3
```
((ActiveMQConnection)connection).setUseAsyncSend(true);
```

#### 异步发送如何确认发送成功?
异步发送时，如果MQ突然宕机，此时生产者端内存中尚未被发送至MQ的消息都会丢失。
所以，正确的异步发送方法是==需要接收回调的==。
```
public class JmsProducer {

    private static final String ACTIVE_MQ_URL = "tcp://49.235.209.207:61616";
    private static final String QUEUE_NAME = "queue01-nio";

    public static void main(String[] args) throws JMSException {
        ActiveMQConnectionFactory activeMQConnectionFactory = new ActiveMQConnectionFactory(ACTIVE_MQ_URL);
        // 开启异步投递
        activeMQConnectionFactory.setAlwaysSyncSend(true);
        Connection connection = activeMQConnectionFactory.createConnection();
        connection.start();

        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);

        Queue queue = session.createQueue(QUEUE_NAME);
        ActiveMQMessageProducer activeMQMessageProducer = (ActiveMQMessageProducer) session.createProducer(queue);

        for(int i=1; i<=3;i++){
            TextMessage textMessage = session.createTextMessage("msg---" + i);
            //设置消息的id
            textMessage.setJMSMessageID(UUID.randomUUID().toString()+"---id");
            String msgId = textMessage.getJMSMessageID();
            // 使用带有回调函数的异步发送
            activeMQMessageProducer.send(textMessage, new AsyncCallback() {
                @Override
                public void onSuccess() {
                    System.out.println(msgId+" success send");
                }

                @Override
                public void onException(JMSException exception) {
                    System.out.println(msgId+" success failed");
                }
            });
        }

        System.out.println("消息发布到MQ成功!");
        activeMQMessageProducer.close();
        session.close();
        connection.close();
    }
}
```

#### [延迟投递和定时投递](http://activemq.apache.org/delay-and-schedule-message-delivery)


物业名称 | 类型 | 描述
--- | --- | ---
AMQ_SCHEDULED_DELAY	| long |	消息在计划由代理传递之前等待的时间（以毫秒为单位）
AMQ_SCHEDULED_PERIOD | long |	在再次计划消息之前等待的开始时间之后等待的时间（以毫秒为单位）
AMQ_SCHEDULED_REPEAT | INT | 重复安排邮件以进行传递的次数
AMQ_SCHEDULED_CRON | String | 使用Cron条目设置计划
1. 开启
```
<broker xmLns=”http://activemq. apache . org/ schema/ core" brokerName= "localhost”dataDirectory= "$ {activemq. data”schedulerSupport="true">
```
2. 
- 要在60秒内安排发送消息
```
MessageProducer producer = session.createProducer(destination);
TextMessage message = session.createTextMessage("test msg");
long time = 60 * 1000;
message.setLongProperty(ScheduledMessage.AMQ_SCHEDULED_DELAY, time);
producer.send(message);
```
- 可以将消息设置为等待初始延迟，重复传递10次，每次重新传递之间等待10秒
```
MessageProducer producer = session.createProducer(destination);
TextMessage message = session.createTextMessage("test msg");
long delay = 30 * 1000;
long period = 10 * 1000;
int repeat = 9;
message.setLongProperty(ScheduledMessage.AMQ_SCHEDULED_DELAY, delay);
message.setLongProperty(ScheduledMessage.AMQ_SCHEDULED_PERIOD, period);
message.setIntProperty(ScheduledMessage.AMQ_SCHEDULED_REPEAT, repeat);
producer.send(message);
```
- 使用CRON来安排消息，例如，如果您希望每小时安排一条消息，则需要将CRON条目设置为 - 0 * * * *
```
MessageProducer producer = session.createProducer(destination);
TextMessage message = session.createTextMessage("test msg");
message.setStringProperty(ScheduledMessage.AMQ_SCHEDULED_CRON, "0 * * * *");
producer.send(message);
```
- 将消息传递10次，每条消息之间有一秒钟的延迟 - 并且您希望每小时发生一次
```
MessageProducer producer = session.createProducer(destination);
TextMessage message = session.createTextMessage("test msg");
message.setStringProperty(ScheduledMessage.AMQ_SCHEDULED_CRON, "0 * * * *");
message.setLongProperty(ScheduledMessage.AMQ_SCHEDULED_DELAY, 1000);
message.setLongProperty(ScheduledMessage.AMQ_SCHEDULED_PERIOD, 1000);
message.setIntProperty(ScheduledMessage.AMQ_SCHEDULED_REPEAT, 9);
producer.send(message);
```

####  [ActiveMQ消费重试机制](http://activemq.apache.org/redelivery-policy)
1. 具体哪些情况会引起消息重发
- Client用了transactions且在session中调用了ollback()
- Client用 了transactions且在调用commit()之前关闭或者没有commit
- Client在CLIENT_ ACKNOWLEDGE的传递模式下，在session中 调用了recover()
2. 请说说消息重发时间间隔和重发次数吗?
- 时间：6
- 间隔：1
3. 有毒消息PoisonACK谈谈你的理解
个消息被redelivedred超过默认的最大重夏次数(默认6次)时，消费端会给MQ发送一 .个”poison ack"表示这个消息有毒，告诉broker不要重发了。这个时候broker会把这个消息放到DLQ(死信队列)
4. 重发参数:   
4.1  collisionAvoidanceFactor: 设置防止冲突范围的正负百分比，只有启用useCollisionAvoidance参数时
才生效。也就是在延迟时间上再加.个时间波动范围。默认值为0.15   
4.2: maximumRedeliveries: 最大重传次数，达到最大重连次数后抛出异常。为-1时不限制次数，为0时表示不进行重传。默认值为6。
4.3: maximumRedeliveryDelay: 最大传送延迟，只在useExponentialBackOff为true时有效 (V5.5)，假设首
次重连间隔为10ms,倍数为2，那么第二次重连时间间隔为20ms,第三次重连时间间隔为40ms，当重连
时间间隔大的最大重连时间间隔时，以后每次重连时间间隔都为最大重连时间间隔。默认为-1。  
4.4: initialRedeliveryDelay: 初始重发延迟时间，默认1000L  
4.5: redeliveryDelay: 重发延迟时间，当initialRedeliveryDelay=0时生效， 默认1000L  
4.6: useCollisionAvoidance: 启用防止冲突功能，默认false   
4.7: useExponentialBackOff:启用指数倍数递增的方式增加延迟时间，默认false   
4.8: backOffMultiplier: 重连时间间隔递增倍数，只有值大于1和启用useExponentialBackOff参数时才生效。默认是5
5. 整合spring，重发配置
```
<!--定义ReDelivery(重发机制)机制-->
<bean id="activeMQRedeliveryPolicy" class="org. apache . activemq . RedeliveryPolicy">
<!--是否在每次尝试重新发送失败后，增长这个等待时间-->
<property name="useExponentialBackOff" value="true" ></property>
(
<!--重发次数,默认为6次这里设置为3次-->
<property name="maximumRedeliveries" value="3"></property>
<!--重发时间间隔,默认为1秒-->
<property name=" initialRedeliveryDelay" value=" 1000></ property>
<!--第一次失败后重新发送之前等待500毫秒,第二次失败再等待500 * 2毫秒,这里的2就是value -->
<property name= "backoffMultiplier" value="2"></ property>
<!--最大传送延迟，
只在useExponertialBackOff为true时有效(V5.5) , 
|
假设苜次重连间隔为10ms倍数为2，那么第二次重连时间间隔为20ms, 第三次重连时间间隔为40ms，
当重连时间间隔大的最大重连时间间隔时，以后每次重连时间间隔都为最大重连时间间隔。- ->
<property name=" maximumRedeliveryDelay" value="1000"></property>
</bean>
```

#### 死信队列
- ActiveMQ中引入了==“死信队列”(Dead Letter Queue)==的概念。即一 条消息再被重发了多次后(默认为重发6次redeliveryCounter==6)，将会被ActiveMQ移入“死信队列”。开发人员用以在这个Queue中查看处理出错的消息，进行==人工干预==。
- 死信队列的使用:处理失败的消息
- SharedDeadLetterStrategy(共享死信队列--默认）
```
<deadLetterStrategy>
<shareDeadLetterStrategy deadl etterQueue="DLQ-QUEUE*/>
</deadLetterStrategy>
```
- IndividualDeadLetterStrategy(独立死信队列):   
使用“queuePrefix” “ topicPrefix"来指定上述前缀。
```
<policyEntry queue="order">
<deadLetterStrategy>
<individualDeadLetterStrategy queuePrefix="DLQ." useQueueForQueueMessages="false" 1>
</deadLetterStrategy>
</policyEntry>
```
将队列Order中出现的DeadLetter保存在DLQ.Order中

- 自动删除过期消息
有时需要直接删除过期的消息而不需要发送到死队列中，“ processExpired”表示是否将过期消息放入死信队列，默认为true;
```
<policyEntry queue= "> ">
    <deadLetterStrategy>
        <sharedDeadLetterStrategy processExpired= "false" />
    </deadL etterStrategy>
</policyEntry>
```

- 存放非持久消息到死信队列
默认情况下，Activemq 不会把非持久的死消息发送到死信队列中I
processNonPersistent”表示是否将“非持久化”消息放入死信队列，默认为false。
非持久性如果你想把非持久的消息发送到死队列中，需要设置属性processNonPersistent=“true ”
```
<policyEntry queue= "> ">
    <deadLetterStrategy>
        <sharedDeadLetterStrategy processNonPersistent= "true" />
    </deadLetterStrategy>
</policyEntry>
```

#### 如何保证消息不被重复消费呢?幂等性问题你谈谈
- 网络延迟传输中，会造成进行MQ重试中，在重试过程中，可能会造成重复消费。
- == 如果消息是做数据库的插入操作，给这个消息做-一个唯一主键， 那么就算出现重复消费的情况，就会导致主键冲突，避免数据库出现脏数据。==
- 如果上面两种情况还不行，准备-一个 第三服务方来做消费记录。以redis为例， 给消息分配-一个全局id, 只要消费过该消息，将<id,message>以
K-V形式写入redis.那消费者开始消费前，先去redis中查询有没消费记录即可。

