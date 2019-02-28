#### 手动安装包

```shell
mvn install:install-file -Dfile={jar_file} -DgroupId={group_id} -DartifactId={artifact_id} -Dversion={version} -Dpackaging=jar
```

#### 自定义包名

```xml
<build>
    <finalName>ROOT</finalName>
</build>
```

#### 复制依赖库

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-dependency-plugin</artifactId>
    <executions>
        <execution>
            <id>copy</id>
            <phase>package</phase>
            <goals>
                <goal>copy-dependencies</goal>
            </goals>
            <configuration>
                <outputDirectory>
                    ${project.build.directory}/lib
                </outputDirectory>
            </configuration>
        </execution>
    </executions>
</plugin>
```

#### 发布项目

##### 使用项目配置进行发布

pom文件配置

```xml
<distributionManagement>
    <snapshotRepository>
        <id>nexus-snapshots</id>
        <url>
            http://172.18.39.121:8081/repository/maven-snapshots/
        </url>
    </snapshotRepository>
</distributionManagement>
```

命令

```shell
mvn deploy
```

##### 通过运行参数进行发布

```shell
mvn deploy -DaltDeploymentRepository=nexus-snapshots::default::http://172.18.31.204:8081/repository/maven-snapshots
```

#### 查看依赖树

```shell
mvn dependency:tree
```

#### 注意

+ 远程仓库没有相应的包时（或者因为网络问题无法下载），在本地也会生成一个pom文件。后面即使在远程仓库下面存在相应的包，maven也不会去下载。要注意删除本地生成出来的文件！







