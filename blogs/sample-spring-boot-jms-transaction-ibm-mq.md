[Home](https://ngbsn.github.io/)

# How to use Spring @Transactional with @JmsListener for IBM MQ

## Overview

This article shows how to create a Spring JMS Listener. To achieve transactions, we shall be using the Transaction
Management from Spring Framework
and for the messaging system we shall be using IBM MQ.
The following example shows a simple Spring Boot application receiving a message from an IN QUEUE on the JMSListener and
publishing the message
to an OUT QUEUE using JmsTemplate.

## Table of Contents

- Create a simple Spring Boot Application
- Configure the Spring Boot Application to use IBM MQ
- Configure the beans for MQ Connection Factory and Transaction Manager.

## Create a simple Spring Boot Application

Let's start with creating a Spring Boot application.

```java
@SpringBootApplication
public class JMSTransactionExampleApplication {
   public static void main(String[] args) {
      SpringApplication.run(JMSTransactionExampleApplication.class, args);
   }
}
```

## Configure the Spring Boot Application to use IBM MQ

Use the following properties in your application.properties file.

```properties
ibm.hostName=localhost
ibm.queueManager=qm1
ibm.channel=DEV.ADMIN.CONN
ibm.port=1414
ibm.userName=user
ibm.password=passw0rd
ibm.clientId=test
```

## Configure the beans for MQ Connection Factory and Transaction Manager.

```java
@Configuration
public class JMSTransactionExampleConfig {

    private static final Logger logger
            = LoggerFactory.getLogger(Config.class);

    @Value("${ibm.hostName}")
    private String hostName;
    @Value("${ibm.queueManager}")
    private String queueManager;
    @Value("${ibm.channel}")
    private String channel;
    @Value("${ibm.port}")
    private Integer port;
    @Value("${ibm.userName}")
    private String userName;
    @Value("${ibm.password}")
    private String password;
    @Value("${ibm.clientId}")
    private String clientId;

    @Bean
    public DefaultJmsListenerContainerFactory jmsListenerContainerFactory() throws JMSException {
        DefaultJmsListenerContainerFactory factory = new DefaultJmsListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory());
        factory.setDestinationResolver(new DynamicDestinationResolver());
        factory.setSessionTransacted(true);
        factory.setConcurrency("2-10");
        factory.setErrorHandler(throwable -> logger.error(throwable.getMessage()));
        return factory;
    }

    @Bean
    public MQQueueConnectionFactory connectionFactory() throws JMSException {
        final MQQueueConnectionFactory connectionFactory = new MQQueueConnectionFactory();
        connectionFactory.setHostName(hostName);
        connectionFactory.setPort(port);
        connectionFactory.setChannel(channel);
        connectionFactory.setQueueManager(queueManager);
        connectionFactory.setAppName(clientId);
        connectionFactory.setStringProperty(USERID, userName);
        connectionFactory.setStringProperty(PASSWORD, password);
        connectionFactory.setTransportType(WMQ_CM_CLIENT);
        connectionFactory.setIntProperty(WMQ_TARGET_CLIENT, WMQ_TARGET_DEST_MQ);
        return connectionFactory;
    }

    @Bean
    public CachingConnectionFactory cachingConnectionFactory() throws JMSException {
        final CachingConnectionFactory cachingConnectionFactory = new CachingConnectionFactory();
        cachingConnectionFactory.setTargetConnectionFactory(connectionFactory());
        cachingConnectionFactory.setCacheConsumers(false);
        cachingConnectionFactory.setCacheProducers(true);
        cachingConnectionFactory.setSessionCacheSize(10);
        cachingConnectionFactory.setClientId("test");
        cachingConnectionFactory.setReconnectOnException(true);
        return cachingConnectionFactory;
    }

    @Bean
    public JmsTemplate jmsTemplate() throws JMSException {
        JmsTemplate jmsTemplate = new JmsTemplate();
        jmsTemplate.setConnectionFactory(cachingConnectionFactory());
        jmsTemplate.setExplicitQosEnabled(true);
        jmsTemplate.setDeliveryPersistent(false);
        jmsTemplate.setSessionTransacted(true);
        return jmsTemplate;
    }

    @Bean
    public JmsTransactionManager transactionManager() throws JMSException {
        JmsTransactionManager jmsTransactionManager = new JmsTransactionManager();
        jmsTransactionManager.setConnectionFactory(cachingConnectionFactory());
        return jmsTransactionManager;
    }
}
```

### Write the JMS Producer and a Service to process the incoming message

#### JMS Producer

```java
@Component
public class JMSProducer {
    private static final Logger logger
            = LoggerFactory.getLogger(JMSProducer.class);
    @Autowired
    JmsTemplate jmsTemplate;

    public void publish(final String message){
        jmsTemplate.convertAndSend("DEV.LOCAL.TEST_OUT_QUEUE", message);
        logger.info("Successfully published message {}", message);
    }
}
```

#### Service

```java
@Service
public class TestService {
    private static final Logger logger
            = LoggerFactory.getLogger(TestService.class);


    @Autowired
    JMSProducer jmsProducer;

    public void process(final String message){
        jmsProducer.publish(message + "-PROCESSED");
        logger.info("Successfully processed message {}", message);
    }
}
```

### Write the JMS Listener

```java
@Component
public class JMSListener {
    private static final Logger logger
            = LoggerFactory.getLogger(JMSListener.class);

    @Autowired
    private TestService testService;

    @JmsListener(destination = "DEV.LOCAL.TEST_IN_QUEUE")
    @Transactional(rollbackFor = Exception.class)
    public void process(final String message) {
        testService.process(message);
    }
}
```

If no exception occurs, transaction will be committed and the message will be removed from the DEV.LOCAL.TEST_IN_QUEUE.
If can exception occurs, transcation is rolled back and message it not removed from DEV.LOCAL.TEST_IN_QUEUE.

### This concludes our article!

