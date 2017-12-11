title: 解决前端JS与后端数据交互长整型精度失真的问题
date: 2017-12-07
tags:
 - 问题解析
 - java
 - javascript
categories:
 - 问题记录
thumbnail: /images/js.png

---

在项目中采用了twitter开源的snowflake算法的id生成器，生成的id是一个long型的大数，因数值太大，通过json形式传输到前端后，在js解析时，会丢失精度。

<!-- more -->

解决办法：

将长整型的数字转为String类型传输到前端，由前端自己负责类型解析。

- 如果使用的是Jackson工具包：

```java
@JsonSerialize(using= ToStringSerializer.class)
private Long id;
```

- 如果使用Fastjson工具包：

局部配置：
```java
public final static SerializeConfig serializeConfig = new SerializeConfig();
static{
    serializeConfig.put(Long.class, new ObjectSerializer() {
          @Override
      public void write(JSONSerializer serializer, Object object, Object fieldName, Type fieldType, int features)
              throws IOException {
          SerializeWriter out = serializer.getWriter();
          out.writeString(Objects.toString(object, null));
      }
    });
}


protected <T> String getResult(T value) {
    JSONObject json = new JSONObject();
    json.put("state", state.getState());
    json.put("desc", state.getDesc());
    json.put("value", value);
    return JSON.toJSONString(json, serializeConfig, SerializerFeature.DisableCircularReferenceDetect);
}
```
全局配置：
```java
SerializeConfig.getGlobalInstance().put(Long.class, new ObjectSerializer() {
            @Override
            public void write(JSONSerializer serializer, Object object, Object fieldName, Type fieldType, int features)
                    throws IOException {
                SerializeWriter out = serializer.getWriter();

                if (fieldType == long.class || fieldType == Long.class) {
                    out.writeString(Objects.toString(object, null));
                }
            }
        });
```

另一个方式是自己编写JSONSerializer和JSONDeserializer。

```java
public class LongJsonSerializer extends JsonSerializer<Long> {
    @Override
    public void serialize(Long value, JsonGenerator jsonGenerator, SerializerProvider serializerProvider) throws IOException, JsonProcessingException {
        String text = (value == null ? null : String.valueOf(value));
        if (text != null) {
            jsonGenerator.writeString(text);
        }
    }
}
```

```java
public class LongJsonDeserializer extends JsonDeserializer<Long> {
    private static final Logger logger = LoggerFactory.getLogger(LongJsonDeserializer.class);

    @Override
    public Long deserialize(JsonParser jsonParser, DeserializationContext deserializationContext) throws IOException, JsonProcessingException {
        String value = jsonParser.getText();
        try {
            return value == null ? null : Long.parseLong(value);
        } catch (NumberFormatException e) {
            throw new RuntimeException(e);
        }
    }
}
```

```java
@JsonSerialize(using = LongJsonSerializer.class)
@JsonDeserialize(using = LongJsonDeserializer.class)
private Long id;
```
