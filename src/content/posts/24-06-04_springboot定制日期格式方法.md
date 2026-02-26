---
title: springboot定制日期格式方法
published: 2024-06-04
description: springboot定制日期格式方法
tags: [java, spring]
category: 后端
---

### **1. 全局配置（application.properties/yml）**
**适用场景**：统一整个应用的日期格式  
**支持的格式**：`java.util.Date`、`java.time.*(如LocalDateTime)`（需额外配置）

```properties
# 设置全局日期格式（针对JSON序列化）
spring.jackson.date-format=yyyy-MM-dd HH:mm:ss
spring.jackson.time-zone=GMT+8  # 设置时区

# 针对Java 8日期类型（如LocalDateTime）需关闭时间戳格式
spring.jackson.serialization.write-dates-as-timestamps=false
```

---

### **2. 使用`@JsonFormat`注解**
**适用场景**：针对特定字段定制序列化格式  
**支持类型**：`Date`、`Calendar`、`java.time.*`

```java
public class User {
    @JsonFormat(pattern = "yyyy/MM/dd", timezone = "Asia/Shanghai")
    private Date birthDate;

    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime createTime;
}
```
> 序列化指将前端传来的json对象转化成java对象
>
> 反序列化则相反
---

### **3. 使用`@DateTimeFormat`注解**
**适用场景**：处理表单提交或`@RequestParam`的参数绑定（**仅反序列化**）  
**支持类型**：`Date`、`Calendar`、`java.time.*`

```java
public class Event {
    @DateTimeFormat(pattern = "dd-MM-yyyy")
    private Date eventDate;
}
```

> **注意**：此注解不影响JSON序列化（返回给前端的格式），需配合`@JsonFormat`使用。

---

### **4. 自定义全局`ObjectMapper`**
**适用场景**：完全控制Jackson的日期序列化行为

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalDateTimeSerializer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

@Configuration
public class JacksonConfig {

    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        
        // 配置Java 8日期时间类型序列化
        JavaTimeModule javaTimeModule = new JavaTimeModule();
        javaTimeModule.addSerializer(
            LocalDateTime.class,
            new LocalDateTimeSerializer(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:ss"))
        );
        
        mapper.registerModule(javaTimeModule);
        mapper.setDateFormat(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
        mapper.setTimeZone(TimeZone.getTimeZone("GMT+8"));
        return mapper;
    }
}
```

---

### **5. 实现`Converter`接口**
**适用场景**：处理控制器参数绑定（如`@RequestParam`、`@PathVariable`）

```java
import org.springframework.core.convert.converter.Converter;
import org.springframework.stereotype.Component;

import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;

@Component
public class StringToDateConverter implements Converter<String, Date> {
    private final SimpleDateFormat format = new SimpleDateFormat("yyyy|MM|dd");

    @Override
    public Date convert(String source) {
        try {
            return format.parse(source);
        } catch (ParseException e) {
            throw new IllegalArgumentException("日期格式无效: " + source);
        }
    }
}
```

---

### **6. 自定义`Formatter`**
**适用场景**：同时处理序列化与反序列化（如Thymeleaf页面渲染）

```java
import org.springframework.format.Formatter;
import java.text.ParseException;
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;
import java.util.Locale;

public class LocalDateFormatter implements Formatter<LocalDate> {
    private final DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy/MM/dd");

    @Override
    public LocalDate parse(String text, Locale locale) throws ParseException {
        return LocalDate.parse(text, formatter);
    }

    @Override
    public String print(LocalDate object, Locale locale) {
        return object.format(formatter);
    }
}
```

注册Formatter：
```java
import org.springframework.context.annotation.Configuration;
import org.springframework.format.FormatterRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addFormatter(new LocalDateFormatter());
    }
}
```

---

### 7. ObjectMapper定制器
```java
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalDateTimeSerializer;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.jackson.Jackson2ObjectMapperBuilderCustomizer;
import org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration;
import org.springframework.context.annotation.Bean;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.Date;
import java.util.TimeZone;

@AutoConfiguration(before = JacksonAutoConfiguration.class)
public class JacksonConfig {

    @Bean
    public Jackson2ObjectMapperBuilderCustomizer customizer() {
        return builder -> {
            // 全局配置序列化返回 JSON 处理
            JavaTimeModule javaTimeModule = new JavaTimeModule();
            DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
            javaTimeModule.addSerializer(LocalDateTime.class, new LocalDateTimeSerializer(formatter));
            builder.modules(javaTimeModule);
            builder.timeZone(TimeZone.getDefault());
        };
    }

}
```
### 8. 消息转化器
```java
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.converter.json.Jackson2ObjectMapperBuilder;
import org.springframework.http.converter.json.MappingJackson2HttpMessageConverter;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
 
@Configuration
public class WebConfig implements WebMvcConfigurer {
 
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        Jackson2ObjectMapperBuilder builder = new Jackson2ObjectMapperBuilder()
            .indentOutput(true)
            .dateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"))
            .modules(new JavaTimeModule()); // 添加Java 8日期时间模块支持
 
        converters.add(new MappingJackson2HttpMessageConverter(builder.build()));
    }
}
```

**个人一般使用ObjectMapper定制器**
