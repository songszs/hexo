title: Gradle中配置使用maven仓库
date: 2018-05-31 16:47:33
category: 工具使用
tags: [gradle,maven]
---
### Maven仓库类型
从 Maven 的依赖下载管理角度来看，Maven 仓库的可分为两类，**远程仓库**和**本地仓库**。

本地仓库是电脑硬盘上的一个目录，远程仓库是在指放在远程服务器上电脑硬盘上的 Maven 仓库目录，使用需添加远程仓库的地址，才能正常连接下载依赖。

Maven 的远程仓库分为**中央仓库**和**私服仓库**。中央仓库存放了世界各地用户上传的依赖包，比较出名的是**JCenter** 和 **Maven Central**，开源的第三方依赖一般都会上传到这两个中央仓库，这样我们只用添加这两个中央仓库的链接地址，就可以下载各种我们需要的依赖了。

由于每次构建都需要从中央仓库下载依赖，对于中央仓库的服务器来说，压力很大，间接导致下载速度很慢。并且，上传到中央仓库的依赖都是开源的，只要知道名字，用户就都能下载，虽然开源精神是值得倡导的，但是一些公司内部的核心依赖包，出于一些考虑，无法直接开源。于是，私服仓库就诞生了。我们可以使用专门的 Maven 仓库管理软件来搭建私服，比如：Apache Archiva，Artifactory，Sonatype Nexus。公司使用的是 **Sonatype Nexus**。在公司的局域网，搭建一个 Nexus 仓库，把公司内部不想开源的依赖包上传到私服仓库中，这样下载依赖的需满足两个条件，私服仓库的地址和在公司局域网内，一定程度上有个保密和安全性。

### 仓库的两种类型

Maven中的仓库分为两种，snapshot快照仓库和release发布仓库。snapshot快照仓库用于保存开发过程中的不稳定版本，release正式仓库则是用来保存稳定的发行版本。

快照版本和正式版本的主要区别在于，本地获取这些依赖的机制有所不同。

release版本第一次构建的时候会把该库从远程仓库中下载到本地仓库缓存，如果版本号不变，以后再次构建都不会去访问远程仓库了。

snapshot版本每次构建时都会优先去远程仓库下载最新的。为了兼顾本地缓存，也可以通过配置来决定向远程仓库中更新SNAPSHOT版本的频率。

### Gradle如何使用Maven仓库中的库 

1. 配置需要使用的maven仓库地址

   在全局Gradle文件中设置Maven仓库地址。如果需要用snapshot则也需要配置snapshot库的地址。
```java
    allprojects 
    {    
        repositories {        
            maven
            {url "http://nexus.gowild.net:8181/content/repositories/gowild-tob/" }        
            maven 
            { url "http://nexus.gowild.net:8181/content/repositories/gowild-tob-snapshots/"}
            }
    }
```

1. 使用maven仓库中某个库的release版本

   在需要使用的project的gradle文件中添加对应依赖。依赖描述由三部分组成，由":"隔开。第一部分为groupid，是标示项目创建团体或组织的唯一标志符，通常是域名倒写比如com.gowild.tob；第二部分artifactId 是项目唯一的标示，比如wifi；第三部分为版本号，比如1.0.0。

```java
    dependencies {
        compile fileTree(include: ['*.jar'], dir: 'libs')
        compile 'com.gowild.tob.wifi:wifi:1.0.0'
    }
```

1. 使用maven仓库中某个库的snapshot版本 

    固定规则，在版本号后面加上-SNAPSHOT，其中SNAPSHOT要为大写。

```java
    dependencies {
        compile fileTree(include: ['*.jar'], dir: 'libs')
        compile 'com.gowild.tob.wifi:wifi:1.0.0-SNAPSHOT'
    }
```

1. 使用aar或者jar

    如果maven仓库里面同时有某个项目的aar或者jar文件，要引用其中一个的话，在后面加@jar或者@aar。

```java
    dependencies {
        compile fileTree(include: ['*.jar'], dir: 'libs')
        compile 'com.gowild.tob.wifi:wifi:1.0.0@jar'
    //    compile 'com.gowild.tob.wifi:wifi:1.0.0@aar'
    //    compile 'com.gowild.tob.wifi:wifi:1.0.0-SNAPSHOT@jar'
    }
```

### 配置snapshot库的更新时间
如果项目中引用了snapshot库，而gradle每次构建的时候没有刷新snapshot库，这是因为snapshot有缓存时间。修改project中gradle配置如下：
```java
    //快照版本及时刷新library
    configurations.all {
        resolutionStrategy.cacheChangingModulesFor 0, 'seconds'
    }
```

### Gradle打包并发布库到maven仓库

如果需要将项目打包jar包或aar包发布到maven仓库中，供其他项目使用，参考如下：

1. 在项目根目录创建nexus_maven_upload.gradle文件
```java
  //使用mavenpublish插件
  apply plugin: 'maven-publish'

   //定义一个用于存储jar名称的变量
   ext { cname = 'builtJar.jar' }

   //获取配置
   Properties properties = new Properties()
   properties.load(project.rootProject.file('nexus_maven_upload.properties').newDataInputStream())

   task sourcesJar(type: Jar) {
       from android.sourceSets.main.java.srcDirs
       classifier = 'sources'
   }
   task javadoc(type: Javadoc) {
       failOnError false
       source = android.sourceSets.main.java.sourceFiles
       classpath += project.files(android.getBootClasspath().join(File.pathSeparator))
       classpath += configurations.compile
   }
   task javadocJar(type: Jar, dependsOn: javadoc) {
       classifier = 'javadoc'
       from javadoc.destinationDir
   }

   //解决 JavaDoc 中文注释生成失败的问题
   tasks.withType(Javadoc) {
       options.addStringOption('Xdoclint:none', '-quiet')
       options.addStringOption('encoding', 'UTF-8')
       options.addStringOption('charSet', 'UTF-8')
   }

   task clearJar(type: Delete) {
       println("clearJar=========================>")
       delete "build/libs/${project.cname}"
   }

   task generateJarName {
       def baseName = "gowild"
       def appendix = project.getName()
       def version = properties.getProperty("POM_VERSION")
       def classifier = "release"
       def extension = "jar"
       project.cname = baseName + "-" + appendix + "-" + version + "-" + classifier + "." + extension
       println("cname===============>" + project.cname)
   }

   task buildJar(dependsOn: ['compileReleaseJavaWithJavac'], type: Jar) {
       println("buildJar=========================>")
       archiveName = project.cname
       //需打包的资源所在的路径集
       def srcClassDir = [project.buildDir.absolutePath + "/intermediates/classes/release"]
       //初始化资源路径集
       from srcClassDir
       exclude '**/BuildConfig.class', '**/R.class'
       exclude { it.name.startsWith('R$') }
   }

   publishing {
       publications {
          //发布jar包
        jar(MavenPublication) {
            println("config jar=========================>")
            groupId properties.getProperty("POM_GROUP_ID")
            artifactId properties.getProperty("POM_ATRIFACT_ID")
            version properties.getProperty("POM_VERSION")
            def desc = properties.getProperty("POM_DESC")
            def packaging = properties.getProperty("POM_PACKAGING")

            if (packaging == "jar") {
                //添加jar包
                artifact("$projectDir/build/libs/${project.cname}")
            } else if (packaging == "aar") {
                //添加aar包
                artifact("$buildDir/outputs/aar/${project.getName()}-release.aar")
            }

            //添加源码
            artifact sourcesJar
            artifact javadocJar

            pom.withXml {
                if (packaging == "jar") {
                    asNode().appendNode('packaging', packageing)
                }
                asNode().appendNode('description', desc)

                //逐个添加依赖配置到pom文件
                def dependencies = asNode().appendNode('dependencies')
configurations.getByName("releaseCompileClasspath").getResolvedConfiguration().getFirstLevelModuleDependencies().each {
                    def dependency = dependencies.appendNode('dependency')
                    dependency.appendNode('groupId', it.moduleGroup)
                    dependency.appendNode('artifactId', it.moduleName)
                    dependency.appendNode('version', it.moduleVersion)
                }
            }
        }

    }
       repositories {
           maven {
               def url_snapshots = properties.getProperty("POM_URL_SNAPSHOTS")
               def url_release = properties.getProperty("POM_URL_RELEASE")
               def version = properties.getProperty("POM_VERSION")
               //根据版本名称选择是发布到release还是snapshot库
               if (version.endsWith('-SNAPSHOT')) {
                   url url_snapshots
               } else {
                   url url_release
               }
               //配置上传的用户名和密码
               credentials {
                   username properties.getProperty("nexus.user")
                   password properties.getProperty("nexus.password")
               }
           }
       }
   }

   //配置task依赖
   //打包前先清空
   buildJar.dependsOn('clearJar')
   //打包前先生成名字
   buildJar.dependsOn('generateJarName')
   //发布前先构建jar
   publish.dependsOn('buildJar')
   //发布前先构建release aar
   publish.dependsOn('assembleRelease')
```

2. 在项目根目录创建配置文件nexus_maven_upload.properties
```java
    ## Project maven upload properties.
    ##仓库release url
    POM_URL_RELEASE=
    ##仓库snapshots url
    POM_URL_SNAPSHOTS=
    POM_GROUP_ID=
    POM_ATRIFACT_ID=
    ##如需发布snapshot版本，版本号后面加上名字"-SNAPSHOT"
    POM_VERSION=1.0.0-SNAPSHOT
    ##发布的描述
    POM_DESC=版本描述
    ##发布的账号
    nexus.user=
    ##发布的密码
    nexus.password=
```
3. 在需要使用该脚本的project或者module的gradle配置中添加上传脚本配置
```java
     apply from: "$project.rootDir.path/nexus_maven_upload.gradle"
```
4. 命令行执行gradle publish发布，或者gradle task列表选择publish执行
```java
     gradle publish
```
