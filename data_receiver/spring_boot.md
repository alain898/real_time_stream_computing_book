## Spring Boot

从手机或网页等客户端，通过互联网发送事件到接收服务器，是金融风控场景下常用的数据采集方式之一。
客户端发送过来的事件包含用户属性、行为、生物识别、终端设备信息、网络状况、TCP/IP协议栈等信息。
HTTP/HTTPS协议筑造了整个互联网的基石，也是当前最主要的应用层通信协议。
没有特别必要，我们采用HTTP/HTTPS协议来进行客户端和数据接收服务器之间的数据通信。

确定数据通信协议后，还需要制定事件上报API（应用程序接口）。
以REST（Representational State Transfer，表述性状态转移）风格为代表的API设计方式，提供了相对标准的API设计准则。
依照REST风格，设计事件上报API如下。

```
POST event/
{
	"user_id": "u200710918",
	"client_timestamp": "1524646221000",
	"event_type": "loan",
	"amount": 1000,
	"...": "..."
}
```

上面的REST API表示向服务器上报一个事件，
用户账号`user_id`是`u200710918`，
发送时间戳`client_timestamp`是`1524646221000`，
事件类型`event_type`是`loan`，
金额`amount`是`1000`，
其它信息用`...`表示。

至此通信协议和API都确定了，接下来实现接收服务器。

### Spring Boot
说到REST风格Web服务器开发，大部分Java编程开发者首先想到的是Spring系列中的Spring Boot。
毫无疑问，Spring Boot使得用Java做Web服务开发的体验相比过去有了极大的提升。
几乎在数分钟之内，一个可用的Web服务就可以开发完毕。
所以我们也先用Spring Boot来实现数据接收服务器，具体实现如下。

```
@Controller
@EnableAutoConfiguration
public class SpringDataCollector {
    private static final Logger logger = LoggerFactory.getLogger(SpringDataCollector.class);

    private JSONObject doExtractCleanTransform(JSONObject event) {
        // TODO: 实现抽取、清洗、转化具体逻辑
        return event;
    }

    private final String kafkaBroker = "127.0.0.1:9092";
    private final String topic = "collector_event";
    private final KafkaSender kafkaSender = new KafkaSender(kafkaBroker);

    @PostMapping(path = "/event", produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
    @ResponseBody()
    public String uploadEvent(@RequestBody byte[] body) {
        // step1: 对消息进行解码
        JSONObject bodyJson = JSONObject.parseObject(new String(body, Charsets.UTF_8));

        // step2: 对消息进行抽取、清洗、转化
        JSONObject normEvent = doExtractCleanTransform(bodyJson);

        // step3: 将格式规整化的消息发到消息中间件kafka
        kafkaSender.send(topic, normEvent.toJSONString().getBytes(Charsets.UTF_8));
        
        // 通知客户端数据采集成功
        return RestHelper.genResponse(200, "ok").toJSONString();
    }

    public static void main(String[] args) throws Exception {
        SpringApplication.run(SpringDataCollector.class, args);
    }
}
```

***说明*** 为了节省篇幅，本书中的样例代码均只保留了主要逻辑以阐述问题，大部分都略去了异常处理和日志打印。
如需将这些代码用于真实产品环境，需要读者自行添加异常处理和日志打印相关内容。
异常处理和日志打印是可靠软件的重要因素，在编程开发时务必重视这两点。

上面的示例代码中，`uploadEvent`实现了事件上报接口。
收到上报事件后，首先对数据进行解码，解码结果用FastJson中的通用JSON类JSONObject表示。
然后在JSONObject对象基础上进行抽取、清洗和转化，规整为统一格式数据。
最后将规整好的数据发往数据传输系统Kafka。

### 总结
本节用Spring Boot开发了数据接收服务器。
从开发体验来看，使用Spring Boot十分便捷，因为Spring Boot隐藏了大量底层的复杂性。
但如果对这些复杂性没有一点了解，数据接收服务器在实际生产上线后可能会出现各种问题。
接下来我们将讨论实现数据接收服务器时可能遇到的各种问题。
