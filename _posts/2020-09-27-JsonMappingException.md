---
title:  "Lombok 으로 선언한 Model 이 Controller 를 통해 생성되지 못 하는 경우"
excerpt: ""

categories:
  - development
tags:
  - development, java, lombok, spring, JsonMappingException, jackson, databind, No, suitable, constructor
last_modified_at: 
  - 2020-09-27T16:16:00+09:00
---

수년 전 Lombok 으로 Getter/Setter 지옥을 벗어나게 되고,
Immutable 에 눈을 뜨기 시작 했었다.

```
import lombok.Builder;
import lombok.Value;

@Builder
@Value
public class InfoModel {
    String id;
    String name;
}
```

그런데, @Value 와 @Builder 를 적용하면서 부터,
한번에 잘 동작하는 곳도 있었지만
Controller 를 통해 Model Instance 가 생성되지 못 하는 프로젝트가 종종 발생했다.

```
- Case1
com.fasterxml.jackson.databind.exc.InvalidDefinitionException: Cannot construct instance of `simple type, class com.seohyang.test.controller.InfoModel` (no Creators, like default construct, exist): cannot deserialize from Object value (no delegate- or property-based Creator)
```

```
- Case2
No suitable constructor found for type [simple type, class com.seohyang.test.controller.InfoModel]: can not instantiate from JSON object (missing default constructor or creator, or perhaps need to add/enable type information?)
 at [Source: {"id":"a","name":"b"}; line: 1, column: 2]input json = {"id":"a","name":"b"}
com.fasterxml.jackson.databind.JsonMappingException: No suitable constructor found for type [simple type, class com.seohyang.test.controller.InfoModel]: can not instantiate from JSON object (missing default constructor or creator, or perhaps need to add/enable type information?)
 at [Source: {"id":"a","name":"b"}; line: 1, column: 2]
	at com.fasterxml.jackson.databind.JsonMappingException.from(JsonMappingException.java:148)
	at com.fasterxml.jackson.databind.deser.BeanDeserializerBase.deserializeFromObjectUsingNonDefault(BeanDeserializerBase.java:1106)
	at com.fasterxml.jackson.databind.deser.BeanDeserializer.deserializeFromObject(BeanDeserializer.java:296)
	at com.fasterxml.jackson.databind.deser.BeanDeserializer.deserialize(BeanDeserializer.java:133)
	at com.fasterxml.jackson.databind.ObjectMapper._readMapAndClose(ObjectMapper.java:3736)
	at com.fasterxml.jackson.databind.ObjectMapper.readValue(ObjectMapper.java:2745)
```

jackson.databind 가 Instance 생성을 위해 사용 할
constructor 를 찾지 못해 발생하는 문제이다.

@Builder, @Value 대신 @Data 를 쓰면 문제는 없어 지지만,
Immutable 을 포기 할 수 없었다.

해결책을 검색 해 보면, lombok.config 파일을 만들고 아래 설정을 선언 하는것이 대부분이다.
lombok.anyConstructor.addConstructorProperties = true

문제는 저 해결책이 잘 작동하는 프로젝트가 있었고
적용해도 해결이 안 되는 프로젝트가 있었다.
(현재 근무중인 조직은 30명이 120개 모듈을 관리한다)

다른 방법을 찾은것이 아래 방법이고, 잘 동작했다.
하지만, Lombok Model 을 생성 할 때마다 행사코드가 많아지고
근본적인 원인을 찾지 못해 마음이 불편했다.
```
- Sol2 : 빌더를 직접 작성 해 준다.

import com.fasterxml.jackson.databind.annotation.JsonDeserialize;
import com.fasterxml.jackson.databind.annotation.JsonPOJOBuilder;

import lombok.Builder;
import lombok.Value;

@JsonDeserialize(builder = InfoModel.InfoModelBuilder.class)
@Builder
@Value
public class InfoModel {
    @JsonPOJOBuilder(withPrefix = "") // https://blog.d46.us/java-immutable-lombok/
    public static final class InfoModelBuilder {
    }

    String id;
    String name;
}

```

어느정도 시간이 지난 후, 우연히 해결책을 찾았다.
lombok.anyConstructor.addConstructorProperties = true
해법이 적용 되려면
jackson-databind 가 2.8.0 이상이여야 한다.

jackson 버젼을 올릴 수 없는 상황이라면
Sol2. https://blog.d46.us/java-immutable-lombok/
에 설명된 방법을 사용 할 수도 있지만,
번거로워서 lombok 을 사용하는 의미가 옅어진다. 