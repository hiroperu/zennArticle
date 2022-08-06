---
title: "SpringでWeb APIのレスポンスJSON内のnullを空文字に変換する"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Java", "Spring"]
published: true
---
（この記事は,筆者の個人ブログで掲載していた内容の移設です）
# やりたいこと
Spring MVCでHttpMessageConverterを通してレスポンスボディに設定されるJSONで値が入っていない(null)際に空文字（""）に自動で変換してもらいたい。
こんなイメージ

```json
  {
      hoge: "hoge",
      fuga: null
  }
```
これを
```json
  {
      hoge: "hoge",
      fuga: ""
  }
```
こうしたい。

fugaはnullなのですが、空文字（""）にしたい。また、別のBeanをレスポンスにマッピングする際も同様の設定で同じような挙動をさせたい。
つまり、webAPI全体での挙動としたい。

# 設定するためにやること
大まかに次の通りです。


1. org.springframework.http.converter.json.Jackson2ObjectMapperBuilderの拡張
2. org.springframework.http.converter.json.Jackson2ObjectMapperFactoryBeanの拡張
3. NullValueSerializerクラスの実装
4. spring-rest-mvc.xmlの編集

spring-rest-mvc.xmlに以下の様に設定されていて、これらがレスポンスにBean(DTOともいう)をマッピングしているのでいじってやります。

```xml
<bean id="jsonMessageConverter"
	class="org.springframework.http.converter.json.MappingJackson2HttpMessageConverter">
	<property name="objectMapper" ref="objectMapper" />
</bean>

<bean id="objectMapper" class="jp.go.mlit.motas.etsuran.api.common.Jackson2ObjectMapperFactoryBean"></bean>
```

構造としてはMappingJackson2HttpMessageConverterが使うobjectMapperがJackson2ObjectMapperFactoryBeanとなっているので、
Jackson2ObjectMapperFactoryBeanをいじって、独自実装したシリアライザーを差し込んでいきます。

## シリアライザーの実装
nullを空文字に設定するシリアライザーを実装します。
```java
import org.codehaus.jackson.JsonGenerator;
import org.codehaus.jackson.JsonProcessingException;
import org.codehaus.jackson.map.JsonSerializer;

public static class NullValueSerializer extends JsonSerializer<Object> {
        @Override
        public void serialize(Object t, JsonGenerator jsonGenerator, SerializerProvider sp)
        throws IOException, JsonProcessingException {
          jsonGenerator.writeString("");
        }
}
```

## Jackson2ObjectMapperBuilderの拡張
拡張したいのは、configureメソッドです。このメソッドが呼び出し元となるJackson2ObjectMapperFactoryBeanから呼ばれて、実際にレスポンスにBeanをマッピングする際に利用するObjectMapperの設定を入れ込んでくれます。

次の様に拡張します。

```java
import 実装したNullValueSerializer;

public class MyBuilder extends Jackson2ObjectMapperBuilder {

    @Override
    public void configure(ObjectMapper objectMapper){
        super.configure(objectMapper);
        DefaultSerializerProvider.Impl dsp = new DefaultSerializerProvider.Impl();
        dsp.setNullValueSerializer(new NullValueSerializer());
        objectMapper.setSerializerProvider(dsp);
    }
}
```

## Jackson2ObjectMapperFactoryBeanの拡張
拡張したJackson2ObjectMapperBuilderを利用する様にこいつも書き換えます。
気をつけなければいけないのは、継承元のJackson2ObjectMapperFactoryBean側のメソッドは呼んでほしくないので、全てのメソッドをオーバーライドする必要があります。

親のメソッドを呼ばれたくない理由は、親クラスのメソッド内で利用されるbuilderは拡張前のJackson2ObjectMapperBuilderを利用しているからです。

```java
public class MyMapperFactoryBean extends Jackson2ObjectMapperFactoryBean {

	private final MyBuilder builder = new MyBuilder();

	@Nullable
	private ObjectMapper objectMapper;

	@Override
	public void setObjectMapper(ObjectMapper objectMapper) {
		this.objectMapper = objectMapper;
	}

	@Override
	public void setCreateXmlMapper(boolean createXmlMapper) {
		this.builder.createXmlMapper(createXmlMapper);
	}

	@Override
	public void setFactory(JsonFactory factory) {
		this.builder.factory(factory);
	}

	//中略


	@Override
	public void afterPropertiesSet() {
		if (this.objectMapper != null) {
			this.builder.configure(this.objectMapper);
		}
		else {
			this.objectMapper = this.builder.build();
		}
	}

	@Override
	@Nullable
	public ObjectMapper getObject() {
		return this.objectMapper;
	}

	@Override
	public Class<?> getObjectType() {
		return (this.objectMapper != null ? this.objectMapper.getClass() : null);
	}

	@Override
	public boolean isSingleton() {
		return true;
	}
}
```

afterPropertiesSetメソッドでJackson2ObjectMapperBuilderを拡張したconfigureが呼ばれて、NullValueSerializerがセットされます。

## spring-rest-mvc.xmlの編集
最後に、Bean定義しているspring-rest-mvc.xmlに拡張してあげたクラスを使う様に書いてあげます。
spring-rest-mvc.xmlに以下の様に設定されていて、これらがレスポンスにBean(DTOともいう)をマッピングしているのでいじってやります。

```xml
<bean id="jsonMessageConverter"
	class="org.springframework.http.converter.json.MappingJackson2HttpMessageConverter">
	<property name="objectMapper" ref="objectMapper" />
</bean>

<bean id="objectMapper" class="MyMapperFactoryBean"></bean>
```

## おわりに
かなり力技での設定になってしまったと思います。Jackson2ObjectMapperBuilderを見ると、
serializersというメソットがあるのでこいつをBean定義しているxmlから呼び出してNullValueSerializerをセットできればもっとスマートになると思っています。
ですが、今回はどのようなレスポンスボディの場合でも統一して空文字にしてほしかったので、
JsonSerializer.handledType()に統一的なレスポンスDTOのオブジェクトとなるクラスを返す様にすることも考えたのですが、ちょっとうまくいかなかったです。
Object.classだとUnknown handled type と言われてエラーになってしまうのです。

この辺りもっとスマートにできるよってのがあればコメントいただけるととても嬉しいです。

## 参考にさせてもらったサイト様

[NullValueSerializerの実装まわり](https://qiita.com/Hyuga-Tsukui/items/e64477094f433ffbb9c1)

[configureの拡張内容まわり](https://qiita.com/n_slender/items/7637042630d7b9da1c41)
