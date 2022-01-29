# Debug

### Class path contains multiple SLF4J bindings <a href="#articlecontentid" id="articlecontentid"></a>

多个依赖产生冲突，通过exlusion来解决

```
<dependencies>
        <dependency>
            <groupId>com.qcloud</groupId>
            <artifactId>vod_api</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>slf4j-api</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>slf4j-log4j12</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
</dependencies>
```

