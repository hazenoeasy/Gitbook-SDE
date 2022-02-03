# Mybatis

### **`Invalid bound statement (not found)`  **&#x20;

xml 文件没有在resource文件夹下，所有不会被打包, 移动xml文件到resource的mapper下,通过配置文件方式，让maven加载.

pom.xml

```
<build>
        <resources>
            <resource>
                <directory>src/main/java</directory>
                <includes>
                    <include>**/*.xml</include>
                </includes>
                <filtering>false</filtering>
            </resource>
        </resources>
    </build>
```

&#x20;在application.properties 下配置&#x20;

```
mybatis-plus.mapper-locations=classpath:plus/yuhaozhang/service/edu/mapper/xml/*.xml
```

