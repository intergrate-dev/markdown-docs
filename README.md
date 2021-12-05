# markdown-docs

## gencode
https://blog.csdn.net/liu649983697/article/details/113930493
https://blog.csdn.net/LittleMangoYX/article/details/108214930
https://www.cnblogs.com/huangjinyong/p/11234268.html

renren-security

jeecg-boot-master

mybatis-plus-code-generator-main

renren-security-master

SpringBootCodeGenerator-master


## performance optimization
### JMH
https://github.com/nitsanw/jmh-samples  (JMH plugin)
https://juejin.cn/post/6960487188228210725
https://www.cnblogs.com/imyalost/p/6802173.html
https://www.cnblogs.com/imyalost/p/7062784.html?utm_source=itdadao&utm_medium=referral

https://github.com/WillemJiang/seckill


## JMeter
https://blog.csdn.net/github_27109687/article/details/71968662
https://blog.csdn.net/xiazdong/category_1214648.html
http://servicecomb.apache.org/cn/docs/performance-test-on-seckill-with-jmeter/
https://blog.csdn.net/kdslkd/article/details/78042864
https://github.com/sdcuike/JMeter-jmx-BeanShellCode
https://github.com/yangengzhe/JMeter

swagger2jmx
https://github.com/liuyunlong1229/swagger2jmx-plugin

test demo
https://github.com/hagyao520/JMeter

script
https://github.com/zhangdongxuan0227/jmeter_scripts
https://github.com/gengyt/workspace_jmeter
https://github.com/tinysheepyang/jmeter_java_scripts

docs
https://github.com/langpf1/jmeter



Rocketmq
本地事务
1. 生产者

消息事务监听

import org.apache.rocketmq.client.producer.LocalTransactionState;
import org.apache.rocketmq.client.producer.TransactionListener;
import org.apache.rocketmq.common.message.Message;
import org.apache.rocketmq.common.message.MessageExt;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
 
import java.util.Date;
 
@Service
public class UserLoginTransactionListener implements TransactionListener {
 
    @Autowired
    private UserLoginLoggerMapper userLoginLoggerMapper;
 
    @Autowired
    private UserLoginTransactionMapper userLoginTransactionMapper;
 
 
 
    @Override
    @Transactional
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
 
        //　核心思路，登录日志(业务数据)　与本地事务状态表在同一个事务中
        UserLoginLogger loginLogger = JSON.parseObject(msg.getBody(), UserLoginLogger.class);
        userLoginLoggerMapper.insert(loginLogger);
 
        UserLoginTransaction userLoginTransaction = new UserLoginTransaction();
        userLoginTransaction.setSerialNumber(loginLogger.getSerialNumber());
        userLoginTransaction.setCreateTime(new Date(System.currentTimeMillis()));
        userLoginTransactionMapper.insert(userLoginTransaction);
 
        //模拟异常,判断事务会不会成功
        int a = 1;
        if(a == 1 ) {
            throw  new RuntimeException();
        }
 
        return LocalTransactionState.UNKNOW;
    }
 
    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
 
        UserLoginLogger loginLogger = JSON.parseObject(msg.getBody(), UserLoginLogger.class);
 
        UserLoginTransaction queryModel = new UserLoginTransaction();
        queryModel.setSerialNumber(loginLogger.getSerialNumber());
 
        UserLoginTransaction transaction = userLoginTransactionMapper.selectOne(queryModel);
 
        if(transaction != null) { // 如果能查询到，则提交消息
            return LocalTransactionState.COMMIT_MESSAGE;
        }
 
 
        // 如果事务状态表中未查询到，返回事务状态未知，RocketMQ默认在15次回查后将该消息回滚
        return LocalTransactionState.UNKNOW;
    }
}
发送消息

@Service
public class UserServiceImpl implements UserService {
 
    @Autowired
    private UserMapper userMapper;
 
    @Autowired
    private UserLoginLoggerMapper userLoginLoggerMapper;
 
    @Autowired
    private TransactionMQProducerContainer transactionMQProducerContainer;
 
    @Autowired
    private UserLoginTransactionListener userLoginTransactionListener;
 
 
    @Override
    public Map<String, Object> login(String userName, String password) {
 
        Map<String, Object> result = new HashMap<>();
        if(StringUtils.isEmpty(userName) || StringUtils.isEmpty(password)) {
            result.put("code", 1);
            result.put("msg", "用户名与密码不能为空");
            return result;
        }
        try {
 
            User queryModel = new User();
            queryModel.setUsername(userName);
            User user = userMapper.selectOne(queryModel);
 
 
            if(user == null || !password.equals(user.getPassword()) ) {
                result.put("code", 1);
                result.put("msg", "用户名或密码不正确");
                return result;
            }
            //登录成功，记录登录日志
            UserLoginLogger userLoginLogger = new UserLoginLogger(user.getId(), new Date(System.currentTimeMillis()));
            userLoginLogger.setSerialNumber( UUID.randomUUID().toString().replace("-",""));
 
            //需要送积分
            if(enbaleActivity()) { //判断是否开启送积分活动，通常可以配置在apollo等配置中心中
                transactionMQProducerContainer.getTransactionMQProducer("user_login_topic", userLoginTransactionListener).sendMessageInTransaction(new Message("user_login_topic", JSON.toJSONString(userLoginLogger).getBytes()), null);
            }
            result.put("code", 0);
            result.put("data", user);
        } catch (Throwable e) {
            e.printStackTrace();
            result.put("code", 1);
            result.put("msg" , e.getMessage());
        }
        return result;
    }
 
 
    private boolean enbaleActivity() {
        return true;
    }
}
生产者（支持事务消息）

public class TransactionMQProducerContainer {
 
    private Map<String, TransactionMQProducer> containers = new HashMap<>();
 
    /**
     * 生产者的组名
     */
    @Value("${apache.rocketmq.producer.producerGroup}")
    private String producerGroup;
 
 
    @Value("${apache.rocketmq.producer.transactionProducerGroup}")
    private String transactionProducerGroup;
 
 
    @Value("${apache.rocketmq.producer.namesrvAddr}")
    private String namesrvAddr;
 
 
 
    public TransactionMQProducer getTransactionMQProducer(String topic, TransactionListener transactionListener) {
 
        TransactionMQProducer producer = containers.get(topic);
        if(producer == null ) {
            synchronized (TransactionMQProducerContainer.class) {
                producer = containers.get(topic);
                if(producer == null ) {
                    producer = new TransactionMQProducer(transactionProducerGroup);
                    producer.setNamesrvAddr(namesrvAddr);
                    producer.setTransactionListener(transactionListener);
                    try {
                        producer.start();
                    }catch (Throwable e) {
                        e.printStackTrace();
                        throw new RuntimeException("生产者启动异常", e);
                    }
 
                    containers.put(topic, producer);
                    return producer;
                }
            }
        }
        return producer;
    }
 
 
}
其他参考
深入理解RocketMQ事务源码--TransactionMQProducer、TransactionListener_向着高亮的地方-CSDN博客  
https://blog.csdn.net/y532798113/article/details/111285548

RocketMQ 实战(五) - 批量消息和事务消息_JavaEdge全是干货的技术号-CSDN博客
https://blog.csdn.net/qq_33589510/article/details/103003721

RocketMQ事务消息实战_中间件兴趣圈-CSDN博客_rocketmq事务消息
https://blog.csdn.net/prestigeding/article/details/81318980

RocketMQ与MYSQL事务消息整合 - zygfengyuwuzu - 博客园

https://blog.csdn.net/y532798113/article/details/111285548

最佳实践

RocketMQ 实战(六) - 最佳实践_JavaEdge全是干货的技术号-CSDN博客
https://javaedge.blog.csdn.net/article/details/103004105

实战RocketMQ解决分布式事务问题 - 铮铮佼佼的技术博客 - OSCHINA - 中文开源技术交流社区
https://my.oschina.net/yangzhongyu/blog/3102933
 

原理
RocketMQ 实战(六) - 最佳实践_JavaEdge全是干货的技术号-CSDN博客

原文链接：https://blog.csdn.net/yzk0308/article/details/121199206


