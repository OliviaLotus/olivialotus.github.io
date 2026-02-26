---
title: RuoYi-Vue-Plus中数据脱敏功能
published: 2024-06-05
description: RuoYi-Vue-Plus中数据脱敏功能
tags: [java, spring, ruoyi]
category: 后端
---

### 1. 注解标注
- **实体类字段标注**：在实体类的字段上添加 `@Sensitive` 注解，指定脱敏策略和角色权限。例如：
  ```java
  @Data
  public class User {
      @Sensitive(strategy = SensitiveStrategy.PHONE, perms = "system:user:edit")
      private String phonenumber;
  }
  ```
  这里，`phonenumber` 字段被标注为使用 `PHONE` 脱敏策略，并且只有具有 `system:user:edit` 权限的用户才能看到脱敏后的手机号。

- **脱敏注解定义**：
  ```java
  @Retention(RetentionPolicy.RUNTIME)
  @Target(ElementType.FIELD)
  @JacksonAnnotationsInside
  @JsonSerialize(using = SensitiveHandler.class)
  public @interface Sensitive {
      SensitiveStrategy strategy(); // 脱敏策略
      String[] roleKey() default {}; // 角色标识符，多个角色满足一个即可
      String[] perms() default {}; // 权限标识符，多个权限满足一个即可
  }
  ```
- **脱敏策略定义**：
  ```java
  @AllArgsConstructor
  public enum SensitiveStrategy {
      PHONE(DesensitizedUtil::mobilePhone),
      ID_CARD(s -> DesensitizedUtil.idCardNum(s, 3, 4)),
      // 其他策略...

      private final Function<String, String> desensitizer;

      public Function<String, String> desensitizer() {
          return desensitizer;
      }
  }
  ```
  - **作用**：
    - 利用hutool的`DesensitizedUtil`定义了不同字段类型的脱敏规则。
    - 通过 `Function<String, String>` 类型的 `desensitizer` 属性，指定具体的脱敏逻辑。

### 2. Jackson 序列化触发
- **序列化过程**：当 Jackson 序列化 `User` 对象时，会遍历其字段。对于 `phonenumber` 字段，Jackson 会识别到其上的 `@Sensitive` 注解。
- **注解处理**：由于 `@Sensitive` 注解上标注了 `@JacksonAnnotationsInside` 和 `@JsonSerialize(using = SensitiveHandler.class)`，Jackson 会将其视为 Jackson 注解的一部分，并调用 `SensitiveHandler` 类来进行序列化。

### 3. 自定义序列化器 `SensitiveHandler`
- **类定义**：
  ```java
  @Slf4j
  public class SensitiveHandler extends JsonSerializer<String> implements ContextualSerializer {
      private SensitiveStrategy strategy;
      private String[] roleKey;
      private String[] perms;

      @Override
      public void serialize(String value, JsonGenerator gen, SerializerProvider serializers) throws IOException {
          try {
              SensitiveService sensitiveService = SpringUtils.getBean(SensitiveService.class);
              if (ObjectUtil.isNotNull(sensitiveService) && sensitiveService.isSensitive(roleKey, perms)) {
                  gen.writeString(strategy.desensitizer().apply(value));
              } else {
                  gen.writeString(value);
              }
          } catch (BeansException e) {
              log.error("脱敏实现不存在, 采用默认处理 => {}", e.getMessage());
              gen.writeString(value);
          }
      }

      @Override
      public JsonSerializer<?> createContextual(SerializerProvider prov, BeanProperty property) throws JsonMappingException {
          Sensitive annotation = property.getAnnotation(Sensitive.class);
          if (Objects.nonNull(annotation) && Objects.equals(String.class, property.getType().getRawClass())) {
              this.strategy = annotation.strategy();
              this.roleKey = annotation.roleKey();
              this.perms = annotation.perms();
              return this;
          }
          return prov.findValueSerializer(property.getType(), property);
      }
  }
  ```
- **作用**：
  - 继承 `JsonSerializer<String>` 类，用于自定义字符串类型的序列化。
  - 实现 `ContextualSerializer` 接口，用于在序列化过程中获取注解信息。
  - 在 `serialize` 方法中，根据注解指定的脱敏策略和角色权限，对字段值进行脱敏处理。
下面是对 `SensitiveHandler` 类中两个方法的详细讲解：

### 1. `serialize` 方法
- **方法作用**：
  - 这是 `JsonSerializer` 类的核心方法，用于自定义字段的序列化逻辑。
  - 在数据脱敏场景中，该方法负责根据注解指定的脱敏策略和角色权限，对字段值进行脱敏处理。

- **方法逻辑**：
  - **获取脱敏服务**：通过 `SpringUtils.getBean(SensitiveService.class)` 获取 `SensitiveService` 实例。
  - **判断是否需要脱敏**：调用 `sensitiveService.isSensitive(roleKey, perms)` 方法，判断当前用户是否具有脱敏权限。
  - **脱敏处理**：如果用户具有脱敏权限，则根据注解指定的脱敏策略（`strategy.desensitizer()`）对字段值进行脱敏处理，并将脱敏后的值写入 JSON 生成器（`gen.writeString(...)`）。
  - **默认处理**：如果用户不具有脱敏权限，或者脱敏服务不存在，则直接将原始字段值写入 JSON 生成器。
  - **异常处理**：如果获取脱敏服务时发生异常（如 `BeansException`），则记录错误日志，并采用默认处理方式，将原始字段值写入 JSON 生成器。

### 2. `createContextual` 方法
- **方法作用**：
  - 这是 `ContextualSerializer` 接口的方法，用于在序列化过程中获取注解信息，并创建具有上下文的序列化器。
  - 在数据脱敏场景中，该方法负责从字段的 `@Sensitive` 注解中提取脱敏策略和角色权限信息，并将其设置到 `SensitiveHandler` 实例中。

- **方法逻辑**：
  - **获取注解信息**：通过 `property.getAnnotation(Sensitive.class)` 获取字段上的 `@Sensitive` 注解。
  - **判断字段类型**：检查字段类型是否为 `String` 类型（`Objects.equals(String.class, property.getType().getRawClass())`），因为脱敏处理只针对字符串类型的字段。
  - **设置脱敏信息**：如果字段上存在 `@Sensitive` 注解且字段类型为 `String`，则从注解中提取脱敏策略（`annotation.strategy()`）、角色标识符（`annotation.roleKey()`）和权限标识符（`annotation.perms()`），并将这些信息设置到 `SensitiveHandler` 实例中。
  - **返回序列化器**：返回当前 `SensitiveHandler` 实例，以便在序列化过程中使用。
  - **默认序列化器**：如果字段上没有 `@Sensitive` 注解或字段类型不是 `String`，则调用 `prov.findValueSerializer(property.getType(), property)` 获取默认的序列化器。


### 4. 脱敏服务 `SensitiveService`
- **接口定义**：
  ```java
  public interface SensitiveService {
      boolean isSensitive(String[] roleKey, String[] perms);
  }
  ```
- **实现类**：
  ```java
  @Service
  public class SensitiveServiceImpl implements SensitiveService {
      @Override
      public boolean isSensitive(String[] roleKey, String[] perms) {
          if (!LoginHelper.isLogin()) {
            return true;
        }
        boolean roleExist = ArrayUtil.isNotEmpty(roleKey);
        boolean permsExist = ArrayUtil.isNotEmpty(perms);
        if (roleExist && permsExist) {
            if (StpUtil.hasRoleOr(roleKey) && StpUtil.hasPermissionOr(perms)) {
                return false;
            }
        } else if (roleExist && StpUtil.hasRoleOr(roleKey)) {
            return false;
        } else if (permsExist && StpUtil.hasPermissionOr(perms)) {
            return false;
        }

        if (TenantHelper.isEnable()) {
            return !LoginHelper.isSuperAdmin() && !LoginHelper.isTenantAdmin();
        }
        return !LoginHelper.isSuperAdmin();
      }
  }
  ```
- **作用**：
  - 定义了判断是否需要脱敏的方法 `isSensitive`。
  - 根据传入的角色标识符和权限标识符，判断当前用户是否具有脱敏权限。
  - 角色和权限同时存在：如果用户同时具有指定的角色和权限中的任意一个，则返回 false，表示不需要进行脱敏处理。
  - 仅角色存在：如果用户具有指定的角色中的任意一个，则返回false，表示不需要进行脱敏处理。
  - 仅权限存在：如果用户具有指定的权限中的任意一个，则返回 false，表示不需要进行脱敏处理。

### 5. 脱敏流程总结
- **注解标注**：在实体类的字段上添加 `@Sensitive` 注解，指定脱敏策略和角色权限。
- **序列化触发**：Jackson 序列化时识别到 `@Sensitive` 注解，调用 `SensitiveHandler`。
- **权限判断**：`SensitiveHandler` 调用 `SensitiveService` 判断用户是否具有脱敏权限。
- **脱敏处理**：如果用户具有脱敏权限，则根据注解指定的脱敏策略对字段值进行脱敏处理。
- **结果输出**：将脱敏后的字段值输出到 JSON 中。

### 6. 示例
假设有一个用户实体类 `User`，其中包含手机号字段 `phonenumber`，我们希望对手机号进行脱敏处理，并且只有具有 `system:user:edit` 权限的用户才能看到脱敏后的手机号：
```java
@Data
public class User {
    @Sensitive(strategy = SensitiveStrategy.PHONE, perms = "system:user:edit")
    private String phonenumber;
}
```
在上述代码中，`@Sensitive` 注解标注了 `phonenumber` 字段需要进行脱敏处理。当 Jackson 序列化 `User` 对象时，会调用 `SensitiveHandler` 的 `serialize` 方法，根据注解指定的脱敏策略和权限，对 `phonenumber` 字段进行脱敏处理
