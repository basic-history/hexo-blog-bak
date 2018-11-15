
---
title: 发布开源项目到 Sonatype
date: 2018-11-04 17:42:00
author: pleuvoir
img: /images/code.jpg
tags:
  - github
  - maven
categories:
  - 技术
---


本文旨在记录在没有私有 maven 仓库的情况下，如何发布项目到公有仓库。以便在有网络的情况下，使用 maven 坐标获取。


## 一、Sonatype 简介

Sonatype 使用 Nexus 为开源项目提供托管服务。你可以通过它发布快照 (snapshot) 或是稳定版 (release) 到 Maven 中央仓库。我们只要注册一个 Sonatype 的 JIRA 账号、创建一个 JIRA ticket，然后对 POM 文件稍作配置即可。


## 二、步骤

### 1. 注册账号

打开 [https://issues.sonatype.org](https://issues.sonatype.org  "https://issues.sonatype.org ") 注册账号，需要注意的是密码必须超过 12 位，且包含至少一个大写字符，一个小写字符，一个特殊字符，以及不少于三种的不同字符（字符，数字，符号）。描述的有些拗口，简单说就是包含大写字母、小写字符、符号和数字，并且超过 12 位即可。所以密码类似于 `Ff123456789/` 这样的。

### 2. 创建 issue

登录成功后，打开 [https://issues.sonatype.org/secure/CreateIssue.jspa?issuetype=21&pid=10134](https://issues.sonatype.org/secure/CreateIssue.jspa?issuetype=21&pid=10134 "https://issues.sonatype.org/secure/CreateIssue.jspa?issuetype=21&pid=10134")

其中 Summary 可以填写项目名

Description 填写项目介绍

groupId `io.github.pleuvoir` (你的)

Project URL 和 SCM url 可以填写 github 项目对应的访问地址


点击创建后会发起一个 issue，会发现状态变为 `OPEN` 接下来要做的就是耐心的等待，等待官方审核。为了方便后续查看，这里是此次演示使用的查看审核状态地址 [查看审核状态](https://issues.sonatype.org/browse/OSSRH-43847 "https://issues.sonatype.org/browse/OSSRH-43847")。此次演示出乎意料，5 分钟之内被审核通过。如下图示例:

![](https://i.imgur.com/9oiOCvW.png)


可以看到，这是官方给出的评论说明。注意相关提示最后一点，当我们发布第一版 release 时，需要评论一次。


> Permalink 
> central-ossrh Central OSSRH added a comment - 5 minutes ago
> io.github.pleuvoir has been prepared, now user(s) pleuvoir can:

> Deploy snapshot artifacts into repository https://oss.sonatype.org/content/repositories/snapshots
> Deploy release artifacts into the staging repository https://oss.sonatype.org/service/local/staging/deploy/maven2
> Promote staged artifacts into repository 'Releases'
> Download snapshot and release artifacts from group https://oss.sonatype.org/content/groups/public
> Download snapshot, release and staged artifacts from staging group https://oss.sonatype.org/content/groups/staging
> please comment on this ticket when you promoted your first release, thanks


### 3. 修改项目

#### 安装 GPG

用于对需要上传的文件加密和签名，`windows` 环境下载地址 [GPG](http://gpg4win.org/ "http://gpg4win.org/")，下载有些慢，如有需要可从第三方源进行下载

安装完成后在命令行输入 `gpg --gen-key` 命令生成自己的 public key，除了姓名、邮箱、备注外其他都可以使用默认配置，最后需要填写一个 passphase（数字即可比如: `62107872006`），它在后面 mvn release 签名时会用到。

最后控制台会显示如下画面:

![](https://i.imgur.com/66XdZPN.png)

其中 `1FEA509E` 即为需要上报的 key id，如果错过了这个画面，可以使用 `gpg --list-keys` 命令查看 key 内容。

上报 key id 给服务器 `gpg --keyserver hkp://pool.sks-keyservers.net --send-keys [你的 key id]`

需要等待执行完毕，不要随便关闭。上报成功后结束，进行下一个环节项目发布。

可以使用 `gpg --keyserver hkp://pool.sks-keyservers.net --recv-keys [你的 key id]` 查看是否发布成功

#### 修改 pom

```xml
<!-- Configuration for maven central repository -->
	<profiles>
		<profile>
			<id>release</id>
			<build>
				<plugins>
					<plugin>
						<groupId>org.sonatype.plugins</groupId>
						<artifactId>nexus-staging-maven-plugin</artifactId>
						<version>1.6.3</version>
						<extensions>true</extensions>
						<configuration>
							<serverId>oss</serverId>
							<nexusUrl>https://oss.sonatype.org/</nexusUrl>
							<autoReleaseAfterClose>true</autoReleaseAfterClose>
						</configuration>
					</plugin>
					<plugin>
						<groupId>org.apache.maven.plugins</groupId>
						<artifactId>maven-gpg-plugin</artifactId>
						<version>1.6</version>
						<executions>
							<execution>
								<phase>verify</phase>
								<goals>
									<goal>sign</goal>
								</goals>
							</execution>
						</executions>
					</plugin>
					<plugin>
						<groupId>org.apache.maven.plugins</groupId>
						<artifactId>maven-source-plugin</artifactId>
						<version>3.0.1</version>
						<executions>
							<execution>
								<phase>package</phase>
								<goals>
									<goal>jar-no-fork</goal>
								</goals>
							</execution>
						</executions>
					</plugin>
					<plugin>
						<groupId>org.apache.maven.plugins</groupId>
						<artifactId>maven-javadoc-plugin</artifactId>
						<version>3.0.0</version>
						<configuration>
							<failOnError>false</failOnError>
							<doclint>none</doclint>
						</configuration>
						<executions>
							<execution>
								<phase>package</phase>
								<goals>
									<goal>jar</goal>
								</goals>
							</execution>
						</executions>
					</plugin>
				</plugins>
			</build>

			<distributionManagement>
				<snapshotRepository>
					<id>oss</id>
					<url>https://oss.sonatype.org/content/repositories/snapshots/</url>
				</snapshotRepository>
				<repository>
					<id>oss</id>
					<url>https://oss.sonatype.org/service/local/staging/deploy/maven2/</url>
				</repository>
			</distributionManagement>
		</profile>
	</profiles>

	<licenses>
		<license>
			<name>The Apache Software License, Version 2.0</name>
			<url>http://www.apache.org/licenses/LICENSE-2.0.txt</url>
		</license>
	</licenses>

	<!-- 修改这里 -->
	<scm>
		<url>https://github.com/pleuvoir/plugins-support</url>
		<connection>scm:git:https://github.com/pleuvoir/plugins-support.git</connection>
		<developerConnection>scm:git:https://github.com/pleuvoir/plugins-support.git</developerConnection>
		<tag>v${project.version}</tag>
	</scm>

	<!-- 修改这里 -->
	<developers>
		<developer>
			<name>pleuvoir</name>
			<email>pleuvior@foxmail.com</email>
			<url>https://pleuvoir.github.io</url>
		</developer>
	</developers>
```

#### 修改 setting

```xml

<!-- Sonatype 账号 -->
<settings>
  <servers>
    <server>
      <id>oss</id>
      <username>修改这里 your-jira-id</username>
      <password>修改这里 your-jira-pwd</password>
    </server>
  </servers>
</settings>


<!-- 配置 gpg 的签名 -->
<settings>
  <profiles>
    <profile>
      <id>oss</id>
      <activation>
        <activeByDefault>true</activeByDefault>
      </activation>
      <properties>
        <gpg.executable>gpg2</gpg.executable>
        <gpg.passphrase>修改这里 the_pass_phrase</gpg.passphrase>
      </properties>
    </profile>
  </profiles>
</settings>
```

#### 发布

`mvn clean deploy -DskipTests -P release`

### 4. 搜索项目

打开 [https://oss.sonatype.org](https://oss.sonatype.org)，search `io.github.pleuvoir` 即可找到上传的项目信息。

如果是第一次 release，需要回复创建的 issue.

## 三、遇到的问题

```bash
[ERROR]  * No public key: Key with id: (491134901fea509e) was not able to be
located on &lt;a href=http://keys.gnupg.net:11371/&gt;http://keys.gnupg.net:1137
1/&lt;/a&gt;. Upload your public key and try the operation again.
```

原因: 各种 OpenPGP 密钥服务器同步需要一些时间。

解决方法: 如果你知道哪个密钥服务器会被查询，你可以直接在那里上传你的密钥。

`gpg --keyserver hkp://keys.gnupg.net --send-keys [你的 key id]`
`gpg --keyserver hkp://keyserver.ubuntu.com --send-keys [你的 key id]`