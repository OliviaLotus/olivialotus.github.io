---
title: RuoYi-Vue-Plus数据过滤功能
published: 2024-06-05
description: RuoYi-Vue-Plus数据过滤功能
tags: [java, spring, ruoyi]
category: 后端
---

## 一、代码不完整,需结合完整源码

## 二、核心组件职责说明

| 组件 | 类名 | 核心职责 |
|------|------|---------|
| **切面** | DataPermissionAspect | 前置上下文准备和后置资源清理 |
| **拦截器** | PlusDataPermissionInterceptor | 拦截SQL执行，协调权限处理流程 |
| **处理器** | PlusDataPermissionHandler | 解析注解，生成权限过滤条件 |
| **上下文** | DataPermissionHelper | 管理线程级权限上下文 |
| **解析器** | JSqlParser | 解析和修改SQL语句 |
| **枚举** | DataScopeType | 定义不同数据范围的处理规则 |

## 三、详细执行流程分析

### 阶段0：切面设置权限上下文（前置处理）

**执行点**：`DataPermissionAspect.doBefore()`

```java
@Before(value = "@annotation(dataPermission)")
public void doBefore(JoinPoint joinPoint, DataPermission dataPermission) {
    // 将Service方法上的注解存入线程上下文
    DataPermissionHelper.setPermission(dataPermission);
}
```

**关键作用**：
1. 在Service方法执行前捕获`@DataPermission`注解
2. 通过`DataPermissionHelper`将注解设置到线程局部变量
3. 建立当前执行的权限规则上下文

### 阶段0'：切面清理上下文（后置处理）

**执行点**：`doAfterReturning()`和`doAfterThrowing()`

```java
// 正常返回清理
@AfterReturning(pointcut = "@annotation(dataPermission)")
public void doAfterReturning(...) {
    DataPermissionHelper.removePermission();
}

// 异常时清理
@AfterThrowing(value = "@annotation(dataPermission)", throwing = "e")
public void doAfterThrowing(...) {
    DataPermissionHelper.removePermission();
}
```

**关键作用**：
1. 确保无论业务方法成功还是异常，都能清理权限上下文
2. 防止线程池复用导致权限信息泄露
3. 释放ThreadLocal资源避免内存泄漏

### 阶段1：方法调用入口（Mapper层）

当调用带有`@DataPermission`注解的Mapper方法时：

```java
@DataPermission({
    @DataColumn(key = "deptName", value = "dept_id"),
    @DataColumn(key = "userName", value = "user_id")
})
@Select("SELECT * FROM sys_user")
List<SysUser> selectAllUsers();
```

### 阶段2：MyBatis拦截器介入（PlusDataPermissionInterceptor）

**执行方法**：`beforeQuery()`

```java
// PlusDataPermissionInterceptor.java
@Override
public void beforeQuery(Executor executor, MappedStatement ms, 
                        Object parameter, RowBounds rowBounds, 
                        ResultHandler resultHandler, BoundSql boundSql) {
    
    // 1. 检查是否忽略权限
    if (InterceptorIgnoreHelper.willIgnoreDataPermission(ms.getId())) {
        return;
    }
    
    // 2. 验证方法是否需要权限处理
    if (dataPermissionHandler.invalid(ms.getId())) {
        return;
    }
    
    // 3. 委托给处理器修改SQL
    PluginUtils.MPBoundSql mpBs = PluginUtils.mpBoundSql(boundSql);
    mpBs.sql(parserSingle(mpBs.sql(), ms.getId()));
}
```

**关键步骤**：
1. 通过`InterceptorIgnoreHelper`检查是否跳过权限
2. 调用`dataPermissionHandler.invalid()`验证方法是否配置权限
3. 使用`parserSingle()`修改原始SQL

### 阶段3：权限处理器工作（PlusDataPermissionHandler）

**执行方法**：`getSqlSegment()`

```java
// PlusDataPermissionHandler.java
public Expression getSqlSegment(Expression where, 
                               String mappedStatementId, 
                               boolean isSelect) {
    try {
        // 1. 获取当前方法权限配置
        DataPermission dataPermission = getDataPermission(mappedStatementId);
        
        // 2. 获取当前用户(优先从线程上下文)
        LoginUser currentUser = DataPermissionHelper.getVariable("user");
        if (ObjectUtil.isNull(currentUser)) {
            currentUser = LoginHelper.getLoginUser();
            DataPermissionHelper.setVariable("user", currentUser);
        }
        
        // 3. 管理员跳过过滤
        if (LoginHelper.isSuperAdmin() || LoginHelper.isTenantAdmin()) {
            return where;
        }
        
        // 4. 构建数据过滤SQL
        String dataFilterSql = buildDataFilter(dataPermission, isSelect);
        
        // 5. 组合新旧条件
        Expression expression = CCJSqlParserUtil.parseExpression(dataFilterSql);
        ParenthesedExpressionList<Expression> parenthesis = 
            new ParenthesedExpressionList<>(expression);
        
        if (ObjectUtil.isNotNull(where)) {
            return new AndExpression(where, parenthesis);
        } else {
            return parenthesis;
        }
    } catch (JSQLParserException e) {
        // ...异常处理
    } finally {
        DataPermissionHelper.removePermission();
    }
}
```

### 阶段4：构建过滤条件（核心中的核心）

**执行方法**：`buildDataFilter()`

```java
// PlusDataPermissionHandler.java
private String buildDataFilter(DataPermission dataPermission, boolean isSelect) {
    // 1. 确定条件连接符
    String joinStr = isSelect ? " OR " : " AND ";
    if (StringUtils.isNotBlank(dataPermission.joinStr())) {
        joinStr = " " + dataPermission.joinStr() + " ";
    }
    
    // 2. 获取当前用户
    LoginUser user = DataPermissionHelper.getVariable("user");
    
    // 3. 准备SpEL上下文
    NullSafeStandardEvaluationContext context = 
        new NullSafeStandardEvaluationContext(defaultValue);
    context.setBeanResolver(beanResolver);
    
    // 4. 处理权限标识符跳过逻辑
    Map<DataColumn, Boolean> ignoreMap = new HashMap<>();
    for (DataColumn dataColumn : dataPermission.value()) {
        if (StringUtils.isNotBlank(dataColumn.permission()) &&
            CollUtil.contains(user.getMenuPermission(), dataColumn.permission())) 
        {
            ignoreMap.put(dataColumn, Boolean.TRUE);
            continue;
        }
        // ...变量设置
    }
    
    // 5. 遍历用户角色构建条件
    Set<String> conditions = new HashSet<>();
    for (RoleDTO role : user.getRoles()) {
        user.setRoleId(role.getRoleId());
        DataScopeType type = DataScopeType.findCode(role.getDataScope());
        
        // 6. 处理不同数据范围类型
        boolean isSuccess = false;
        for (DataColumn dataColumn : dataPermission.value()) {
            if (ignoreMap.containsKey(dataColumn)) {
                conditions.add(joinStr + " 1 = 1 ");
                isSuccess = true;
                continue;
            }
            
            // 7. SpEL表达式解析
            String sql = DataPermissionHelper.ignore(() ->
                parser.parseExpression(type.getSqlTemplate(), parserContext)
                    .getValue(context, String.class)
            );
            
            conditions.add(joinStr + sql);
            isSuccess = true;
        }
        
        // 8. 兜底方案
        if (!isSuccess && StringUtils.isNotBlank(type.getElseSql())) {
            conditions.add(joinStr + type.getElseSql());
        }
    }
    
    // 9. 合并条件
    if (CollUtil.isNotEmpty(conditions)) {
        String sql = StreamUtils.join(conditions, Function.identity(), "");
        return sql.substring(joinStr.length());
    }
    return StringUtils.EMPTY;
}
```

### 阶段5：SpEL表达式解析（动态SQL生成）

以**部门及以下**数据范围为例：

```java
// DataScopeType.java
DEPT_AND_CHILD("4", 
    "#{#deptName} IN ( #{@sdss.getDeptAndChild( #user.deptId )} )", 
    " 1 = 0 ")
```

**解析过程**：
1. `#user.deptId`：从上下文获取当前用户部门ID
2. `@sdss.getDeptAndChild()`：调用Spring Bean的方法
3. `#{#deptName}`：替换为注解配置的`dept_id`

**最终生成SQL**：
```sql
dept_id IN (SELECT dept_id FROM sys_dept 
            WHERE FIND_IN_SET(dept_id, getDeptChildList(100)))
```

### 阶段6：上下文管理（DataPermissionHelper）

**核心功能**：
```java
// DataPermissionHelper.java
public class DataPermissionHelper {
    // 线程级权限上下文
    private static final String DATA_PERMISSION_KEY = "data:permission";
    
    // 获取变量
    public static <T> T getVariable(String key) {
        Map<String, Object> context = getContext();
        return (T) context.get(key);
    }
    
    // 设置变量
    public static void setVariable(String key, Object value) {
        Map<String, Object> context = getContext();
        context.put(key, value);
    }
    
    // 获取上下文Map
    public static Map<String, Object> getContext() {
        SaStorage saStorage = SaHolder.getStorage();
        Object attribute = saStorage.get(DATA_PERMISSION_KEY);
        // ...初始化处理
        return (Map<String, Object>) attribute;
    }
}
```

## 四、多角色数据范围合并策略

当用户拥有多个角色时，权限处理器采用**并集策略**：

```java
for (RoleDTO role : user.getRoles()) {
    // 处理每个角色的数据范围
    // ...
    
    // 生成的每个角色条件用OR连接
    conditions.add(joinStr + sql);
}

// 最终组合形式
String finalCondition = condition1 + " OR " + condition2;
```

**特殊处理**：
- 如果任一角色有**全部数据权限(ALL)**，直接返回空条件
- 管理员角色跳过所有过滤
- 权限标识符满足时添加`1=1`永真条件
