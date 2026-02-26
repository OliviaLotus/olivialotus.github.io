---
title: springboot常用依赖
published: 2024-06-29
description: springboot常用依赖
tags: [spring, java]
category: 后端
---

```xml

    <properties>
        <revision>1.0.0</revision>
        <java.version>17</java.version>
        <knife4j.version>4.5.0</knife4j.version>
        <fastexcel.version>1.2.0</fastexcel.version>
        <mybatis-plus.version>3.5.12</mybatis-plus.version>
        <mapstruct-plus.version>1.4.8</mapstruct-plus.version>
        <mapstruct-plus.lombok.version>0.2.0</mapstruct-plus.lombok.version>
        <hutool-all.version>5.8.38</hutool-all.version>
        <satoken.version>1.44.0</satoken.version>
        <springdoc.version>2.5.0</springdoc.version>
        <spring-admin.version>3.5.0</spring-admin.version>
        <sms4j.version>3.3.5</sms4j.version>
        <ip2region.version>2.7.0</ip2region.version>
        <redisson.version>3.50.0</redisson.version>
        <lock4j.version>2.2.7</lock4j.version>
        <simple-java-mail.version>8.12.6</simple-java-mail.version>
        <easy-captcha.version>1.6.2</easy-captcha.version>
        <UserAgentUtils.version>1.21</UserAgentUtils.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <!-- 权限认证 -->
            <dependency>
                <groupId>cn.dev33</groupId>
                <artifactId>sa-token-spring-boot3-starter</artifactId>
                <version>${satoken.version}</version>
            </dependency>
            <dependency>
                <groupId>cn.dev33</groupId>
                <artifactId>sa-token-jwt</artifactId>
                <version>${satoken.version}</version>
            </dependency>
            <!-- 数据库 -->
            <dependency>
                <groupId>com.baomidou</groupId>
                <artifactId>mybatis-plus-spring-boot3-starter</artifactId>
                <version>${mybatis-plus.version}</version>
            </dependency>
            <dependency>
                <groupId>com.baomidou</groupId>
                <artifactId>mybatis-plus-jsqlparser</artifactId>
                <version>${mybatis-plus.version}</version>
            </dependency>
            <dependency>
                <groupId>io.github.linpeilie</groupId>
                <artifactId>mapstruct-plus-spring-boot-starter</artifactId>
                <version>${mapstruct-plus.version}</version>
            </dependency>
            <!-- 监控 -->
            <dependency>
                <groupId>de.codecentric</groupId>
                <artifactId>spring-boot-admin-starter-server</artifactId>
                <version>${spring-admin.version}</version>
            </dependency>
            <dependency>
                <groupId>de.codecentric</groupId>
                <artifactId>spring-boot-admin-starter-client</artifactId>
                <version>${spring-admin.version}</version>
            </dependency>
            <!-- 工具 -->
            <dependency>
                <groupId>cn.hutool</groupId>
                <artifactId>hutool-all</artifactId>
                <version>${hutool-all.version}</version>
            </dependency>
            <dependency>
                <groupId>cn.idev.excel</groupId>
                <artifactId>fastexcel</artifactId>
                <version>${fastexcel.version}</version>
            </dependency>
            <!-- 短信 -->
            <dependency>
                <groupId>org.dromara.sms4j</groupId>
                <artifactId>sms4j-spring-boot-starter</artifactId>
                <version>${sms4j.version}</version>
            </dependency>
            <!-- 邮箱 -->
            <dependency>
                <groupId>org.simplejavamail</groupId>
                <artifactId>simple-java-mail</artifactId>
                <version>${simple-java-mail.version}</version>
            </dependency>
            <!-- 验证码 -->
            <dependency>
                <groupId>com.github.whvcse</groupId>
                <artifactId>easy-captcha</artifactId>
                <version>${easy-captcha.version}</version>
            </dependency>
            <!-- 接口文档 -->
            <dependency>
                <groupId>com.github.xiaoymin</groupId>
                <artifactId>knife4j-openapi3-jakarta-spring-boot-starter</artifactId>
                <version>${knife4j.version}</version>
            </dependency>
            <!-- IP设备 -->
            <dependency>
                <groupId>eu.bitwalker</groupId>
                <artifactId>UserAgentUtils</artifactId>
                <version>${UserAgentUtils.version}</version>
            </dependency>
            <dependency>
                <groupId>org.lionsoul</groupId>
                <artifactId>ip2region</artifactId>
                <version>${ip2region.version}</version>
            </dependency>
            <!--分布式-->
            <dependency>
                <groupId>org.redisson</groupId>
                <artifactId>redisson-spring-boot-starter</artifactId>
                <version>${redisson.version}</version>
            </dependency>
            <dependency>
                <groupId>com.baomidou</groupId>
                <artifactId>lock4j-redisson-spring-boot-starter</artifactId>
                <version>${lock4j.version}</version>
            </dependency>


        </dependencies>
    </dependencyManagement>


    <build>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <configuration>
                        <source>${java.version}</source>
                        <target>${java.version}</target>
                        <annotationProcessorPaths>
                            <path>
                                <groupId>org.projectlombok</groupId>
                                <artifactId>lombok</artifactId>
                                <version>${lombok.version}</version>
                            </path>
                            <path>
                                <groupId>io.github.linpeilie</groupId>
                                <artifactId>mapstruct-plus-processor</artifactId>
                                <version>${mapstruct-plus.version}</version>
                            </path>
                            <path>
                                <groupId>org.projectlombok</groupId>
                                <artifactId>lombok-mapstruct-binding</artifactId>
                                <version>${mapstruct-plus.lombok.version}</version>
                            </path>
                            <path>
                                <groupId>org.springframework.boot</groupId>
                                <artifactId>spring-boot-configuration-processor</artifactId>
                            </path>
                        </annotationProcessorPaths>
                    </configuration>
                </plugin>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <configuration>
                        <excludes>
                            <exclude>
                                <groupId>org.projectlombok</groupId>
                                <artifactId>lombok</artifactId>
                            </exclude>
                        </excludes>
                    </configuration>
                </plugin>
                <!-- 统一版本号管理 -->
                <plugin>
                    <groupId>org.codehaus.mojo</groupId>
                    <artifactId>flatten-maven-plugin</artifactId>
                    <configuration>
                        <updatePomFile>true</updatePomFile>
                        <flattenMode>resolveCiFriendliesOnly</flattenMode>
                    </configuration>
                    <executions>
                        <execution>
                            <id>flatten</id>
                            <phase>process-resources</phase>
                            <goals>
                                <goal>flatten</goal>
                            </goals>
                        </execution>
                        <execution>
                            <id>flatten.clean</id>
                            <phase>clean</phase>
                            <goals>
                                <goal>clean</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>
            </plugins>

    </build>
```
