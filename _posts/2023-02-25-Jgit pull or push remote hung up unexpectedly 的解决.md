---
layout: post
title:  "Jgit pull or push remote hung up unexpectedly 的解决"
categories: Java工具
tags:  Java工具
author: zhimaxingzhe
link: https://zhimaxingzhe.github.io/
---

* content
{:toc}

# 问题定义
在上篇中提到通过 Jgit 实现对公司git的操作，达成业务上的一些功能，最近在扩容和部署到新租户环境下时，部分机器遇到操作远程仓库时报会报 remote hung up unexpectedly。
经过一番面向互联网编程后，逐步确认是因为服务器上仓库通过https 模式 clone 下来后被重置为 ssh 模式，目前还不确定是为什么这些机器会有这样的现象。具体报错信息如下：

```java
org.eclipse.jgit.api.errors.TransportException: git@git.xxx.com:/xxxx.git: remote hung up unexpectedly
        at org.eclipse.jgit.api.FetchCommand.call(FetchCommand.java:246)
        at org.eclipse.jgit.api.PullCommand.call(PullCommand.java:266)
............... 省略其他
Caused by: org.eclipse.jgit.errors.TransportException: git@git.xxx.com:/xxxx.git: remote hung up unexpectedly
        at org.eclipse.jgit.transport.TransportGitSsh$SshFetchConnection.<init>(TransportGitSsh.java:313)
        at org.eclipse.jgit.transport.TransportGitSsh.openFetch(TransportGitSsh.java:153)
        at org.eclipse.jgit.transport.FetchProcess.executeImp(FetchProcess.java:151)
        at org.eclipse.jgit.transport.FetchProcess.execute(FetchProcess.java:103)
        at org.eclipse.jgit.transport.Transport.fetch(Transport.java:1462)
        at org.eclipse.jgit.api.FetchCommand.call(FetchCommand.java:235)
```

在于远程仓库交互做身份验证时模式无法匹配，Jgit用https的身份认证方式操作了ssh模式的本地仓库。






# 问题解决
既然是被设置为 ssh 方式，应对方式有两种方式，方法一是将服务器上本地 git 仓库改为 https 模式，方法二是代码中做校验，根据仓库类型再选择对应的认证方式。 

方法一是修改确认clone方式为https，若git clone https导出还是不行，则暴力的方式是修改.git/config配置文件的URL，将ssh路径改为https，或者直接将账户密码配置到URL中（url = https://[git账户]:[git密码]@git.xxx.com/xxx/xxx.git），绕过身份验证。   

方法二的两种认证方式在上一篇《Java通过JGit操作Git的使用方法》中有提到：

HTTP(S)账号+密码：
```java
// 账号 + 密码
UsernamePasswordCredentialsProvider provider = new UsernamePasswordCredentialsProvider("userName-zhimaxingzhe", password);
// 拉取远程更新        
git.pull().setCredentialsProvider(provider).setRemoteBranchName(branchName).call();
```
SSH协议使用公钥认证：
```java
SshSessionFactory sshSessionFactory = new JschConfigSessionFactory() {
    @Override
    protected void configure(OpenSshConfig.Host host, Session session) {
        session.setConfig("StrictHostKeyChecking", "no");
    }

    @Override
    protected JSch createDefaultJSch(FS fs) throws JSchException {
        JSch sch = super.createDefaultJSch(fs);
        //keyPath 公钥文件 path，执行ssh-keygen -o 命令生成，配置到账户信任key中
        sch.addIdentity(keyPath);
        return sch;
    }
};
// 打开本地仓库
Git git = Git.open(new File("/Users/zhimaxingzhe/test"));
// 执行远程拉取合并时使用
git.pull().setTransportConfigCallback(
    transport -> {
        SshTransport sshTransport = ( SshTransport )transport;
        sshTransport.setSshSessionFactory( sshSessionFactory );
    });
```

其实还有一种方法，我没有尝试，但看原文作者是解决了，这里贴出链接供参考[https://juejin.cn/post/7132469765934678030](https://juejin.cn/post/7132469765934678030)

参考资料：   
[jgit-cookbook](https://github.com/centic9/jgit-cookbook)

> by zhimaxingzhe from [Jgit pull or push remote hung up unexpectedly 的解决](https://zhimaxingzhe.github.io/2023/02/25/Jgit-pull-or-push-remote-hung-up-unexpectedly/) 欢迎分享链接，转载请注明出处，尊重版权，若急用请联系授权。 [https://zhimaxingzhe.github.io](https://zhimaxingzhe.github.io)
