

## dockerfile-maven-plugin使用指南

最近在将应用部署到容器平台，需要在打包时生成docker镜像，在网上首先搜到了docker-maven-plugin这个插件，但使用起来很麻烦，在maven和dockfile都要做很多额外的配置。后来在官方Github看到作者推荐使用dockerfile-maven-plugin这个新的插件，于是替换成这个，但这个插件在网上的相关资料较少。在此记录一哈



### pom配置

```xml
<plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-deploy-plugin</artifactId>
                <configuration>
                    <skip>true</skip>
                </configuration>
            </plugin>

            <!-- Dockerfile maven plugin -->
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>dockerfile-maven-plugin</artifactId>
                <version>1.4.10</version>
                <executions>
                    <!--<execution>-->
                        <!--<id>default</id>-->
                        <!--<goals>-->
                            <!--&lt;!&ndash;如果package时不想用docker打包,就注释掉这个goal&ndash;&gt;-->
                            <!--<goal>build</goal>-->
                            <!--<goal>push</goal>-->
                        <!--</goals>-->
                    <!--</execution>-->
                </executions>
                <configuration>
                    <repository>docker-reg.sogou-inc.com/feedback/${artifactId}-${profiles.active}</repository>
                    <tag>${project.version}</tag>
                    <buildArgs>
                        <JAR_FILE>target/${project.build.finalName}.jar</JAR_FILE>
                    </buildArgs>
                </configuration>
            </plugin>
```



### setting.xml配置

这个文件在maven目录下，可以 cd $M2_HOME/conf 进入。

在pluginGroups中增加一个<pluginGroup>com.spotify</pluginGroup>

```xml
  <pluginGroups>
    <pluginGroup>com.spotify</pluginGroup>
  </pluginGroups>
```

### 登录情况

#### 需要登录

关于如何验证登录，坑比较多。如果你在habor设置你的仓库为私有，那必须要登录，按照官方配置就可以，如下。

```xml
 <plugin>
    <groupId>com.spotify</groupId>
    <artifactId>dockerfile-maven-plugin</artifactId>
    <version>${version}</version>
    <configuration>
        <username>repoUserName</username>
        <password>repoPassword</password>
        <repository>${docker.image.prefix}/${project.artifactId}</repository>
        <buildArgs>
            <JAR_FILE>target/${project.build.finalName}.jar</JAR_FILE>
        </buildArgs>
    </configuration>
</plugin>
```



#### 无需登录

但因为我配置了在k8s自动从habor获取镜像，所以设置了公开，这种情况下无需登录，但有时也会执行失败，此时你需要删掉~/.docker/config.json中的这个网站的配置。

```
cat ~/.docker/config.json

{
    "auths": {
        "192.168.87.110:5000": {
            "auth": "YWRtaW46JKDtaW4xMjM="
        }(删掉此处)
    },
    "HttpHeaders": {
        "User-Agent": "Docker-Client/18.09.0 (linux)"
    }
}
```



确认这里为空后，如果还报错，可以再执行一次docker login ... ，这样就成功了

### maven多模块情况配置

在多模块的情况下，打包插件一定要放置在Application子模块中，如果放在root pom中会导致打包不成功。

如下情况：

-app

​	-common

​	-file

​    -mail

​    -application

​    -pom.xml

这种情况，我们可以分两个步骤

第一步先打包全部模块，在根目录下

```shell
mvn clean package -P test
```

第二步在要打包镜像的子模块中执行deploy命令

```shell
mvn dockerfile:build dockerfile:push
```

这样，就能成功将子模块打包为镜像并push。



### jenkins

在本地测试完之后，要将这个流程弄到jenkins，做一些配置。

#### jenkins服务器安装docker

此处不再赘述，maven的setting.xml等配置和本地一样。

#### 修改jenkins项目配置

此时，由于项目需要打包两次（一次在根目录打包，第二次在子目录打包为镜像），所以需要执行两次mvn命令，和之前不一样，所以将第一次的执行还是使用jenkins的Build模块。

[![MkbuLT.md.png](https://s2.ax1x.com/2019/11/07/MkbuLT.md.png)](https://imgchr.com/i/MkbuLT)

第二次的执行放置在post steps中通过命令在执行

[![Mkb3FJ.md.png](https://s2.ax1x.com/2019/11/07/Mkb3FJ.md.png)](https://imgchr.com/i/Mkb3FJ)



```
cd 子模块目录
mvn clean package -P $env dockerfile:build dockerfile:push
```

这样，就可以完成打包并制作镜像的步骤了