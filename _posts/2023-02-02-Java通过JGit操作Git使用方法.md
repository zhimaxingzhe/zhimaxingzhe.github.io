---
layout: post
title:  "Java通过JGit操作Git的使用方法"
categories: Java工具
tags:  Java工具
author: zhimaxingzhe
link: https://zhimaxingzhe.github.io/
---

* content
{:toc}
> by zhimaxingzhe from [Java通过JGit操作Git的使用方法](https://zhimaxingzhe.github.io/2023/02/02/Java通过JGit操作Git的使用方法/) 欢迎分享链接，转载请注明出处，尊重版权，若急用请联系授权。 [https://zhimaxingzhe.github.io](https://zhimaxingzhe.github.io)


# 前言
Java 社区操作与GIT交互的最好组件应该就是JGit了，目前没找到更好的，JGit是实现Git版本控制系统的纯Java库。 这是一个Eclipse项目，最初是EGit的Git库，它提供了与Eclipse的Git集成。 同时，JGit还有更多采用者，例如Gerrit，GitBlit，Jenkins的GitClient插件。近期在做配置文件发布功能时用到了，学习了JGit的使用，做一下分享。若有助益，请一键三连吧🤝。
![封面-宽.png](https://zhimaxingzhe.github.io/image/jgit/封面.png)  

与GIT交互如何做？最早我想通过在服务器上安装git客户端，在Java代码中执行shell命令的方式来实现对git的操作，这样一来非常灵活，代码都写好了（见文末）。但这样与GIT交互开发、调试工作量巨大，且部署需要运维做较多工作，且存在安全问题，对权限的控制要求严格。
后来找到JGit，了解到JGit 可以不依赖服务器安装 git client 即可对 GIT 进行操作，而且对GIT的绝大部分操作都封装好了API，真是太适合我的场景了。

此外，通过Jgit 直接操作Github，做成自己日常的工具，比如写博客，想要定时发布到Github上，则可以通过自己写代码实现啦。



# 一、导入依赖

当前最新的版本是6.4.0版本，而且是JDK 11编译版本，要求JDK 是11及其以上版本才行，如果是JDK 8 得找旧版本的，比如5.13版本，可能有漏洞风险，要视情况来使用。截稿前5.13版本没看到有漏洞风险，可以使用。
```
<dependency>
    <groupId>org.eclipse.JGit</groupId>
    <artifactId>org.eclipse.JGit</artifactId>
    <version>6.4.0.202210260700-m2</version>
</dependency>

<dependency>
    <groupId>org.eclipse.JGit</groupId>
    <artifactId>org.eclipse.JGit</artifactId>
    <version>5.13.1.202206130422-r</version>
</dependency>
```
# 二、基础操作
## 1、创建仓库推送到远程
本地已存在目录初始化为仓库后推送到远程

```shell
cd "/Users/zhimaxingzhe/test"
git init
git remote add origin https://github.com/zhimaxingzhe/test.git
git add .
git commit -m "init"
git push -u origin master
```

```java
// 初始化本地仓库，标记远程仓库地址 "https://github.com/zhimaxingzhe/test.git"
Git git = Git.init().setDirectory(new File("/Users/zhimaxingzhe/test")).call();
StoredConfig config = git.getRepository().getConfig();
config.setString("remote", "origin", "url", "https://github.com/zhimaxingzhe/test.git");
config.save();
git.commit().setMessage("init").call();
//推到远程仓库
UsernamePasswordCredentialsProvider provider =
                    new UsernamePasswordCredentialsProvider("userName-zhimaxingzhe", password);
git.push().setRemote("origin").setCredentialsProvider(provider)
                    .setRefSpecs(new RefSpec("master")).call();

```


远程已存在的仓库

```shell
git clone https://github.com/zhimaxingzhe/test.git
cd test
touch README.md
git add README.md
git commit -m 'add README.md'
git push -u origin master
```
对应的，通过JGit是如何操作的呢？直接上代码
```java
// 克隆远程仓库到本地
Git git = Git.cloneRepository().setURI("https://github.com/zhimaxingzhe/test.git").setTimeout(90)
                .setDirectory(new File("/Users/zhimaxingzhe/test"))
                .setCredentialsProvider(provider).call();
// 创建本地文件
FileUtil.touch("/Users/zhimaxingzhe/test"+ "/README.md");
//添加全部文件
git.add().addFilepattern("README.md").call(); 
//提交
git.commit().setMessage("add README").call();
//推到远程仓库
UsernamePasswordCredentialsProvider provider =
                    new UsernamePasswordCredentialsProvider("userName-zhimaxingzhe", password);
git.push().setRemote("origin").setCredentialsProvider(provider)
                    .setRefSpecs(new RefSpec("master")).call();
```

## 2、 clone 远端仓库到本地
```java
// 提供用户名和密码的验证
UsernamePasswordCredentialsProvider provider = new UsernamePasswordCredentialsProvider("userName", "password");
// 判断本地目录是否存在
boolean exist = FileUtil.exist("/Users/zhimaxingzhe/");
if (!exist) {
    FileUtil.mkdir("/Users/zhimaxingzhe/");
}
// clone 仓库到指定目录
Git git = Git.cloneRepository().setURI(GIT_URL).setDirectory(new File("/Users/zhimaxingzhe/test"))
        .setCredentialsProvider(provider).call();
```

## 3、身份认证
上面的例子中都有用到用户名密码做身份验证，所以顺着讲一下身份认证。在做远程仓库操作是需要身份认证，如执行pull、push、clone操作时。和git交互一样，身份认证JGit 支持两种方式，一种是HTTP(S)账号+密码的方式，一种是通过SSH协议使用公钥认证。  
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
        //keyPath 公钥文件 path
        sch.addIdentity(keyPath);
        return sch;
    }
};
// 打开本地仓库
Git git = Git.open(new File("/Users/zhimaxingzhe/test"));
// 执行远程拉取合并
git.pull().setTransportConfigCallback(
    transport -> {
        SshTransport sshTransport = ( SshTransport )transport;
        sshTransport.setSshSessionFactory( sshSessionFactory );
    });
git.call();
// git push
git.push().setRemote("origin").setTransportConfigCallback(transport -> {
  SshTransport sshTransport = ( SshTransport )transport;
  sshTransport.setSshSessionFactory( sshSessionFactory );
  }).setRefSpecs(new RefSpec(branchName)).call();
```
## 4、拉取更新 git pull
```java
UsernamePasswordCredentialsProvider provider = new UsernamePasswordCredentialsProvider("userName-zhimaxingzhe", password);
// 拉取远程更新        
git.pull().setCredentialsProvider(provider).setRemoteBranchName(branchName).call();
```

## 5、添加文件 git add
添加所有修改文件
 ```java
 //打开git仓库  
Git git = Git.open(new File("/Users/zhimaxingzhe/test"));
git.add().addFilepattern(".").call();
 ``` 
 按文件名称添加文件
 ```java
 //打开git仓库  
Git git = Git.open(new File("/Users/zhimaxingzhe/test"));
git.add().addFilepattern("README.md").call(); 
 ```
 如果知道要添加的文件列表，则可以根据本地仓库文件状态按修改类型进行添加
```java
//打开git仓库  
Git git = Git.open(new File("/Users/zhimaxingzhe/test"));
//判断是否有被修改过的文件  
List<DiffEntry> diffEntries = git.diff()  
    .setPathFilter(PathFilterGroup.createFromStrings(files))  
    .setShowNameAndStatusOnly(true).call();  
if (diffEntries == null || diffEntries.size() == 0) {  
    throw new Exception("提交的文件内容都没有被修改，不能提交");  
}  
//被修改过的文件  
List<String> updateFiles=new ArrayList<String>();  
ChangeType changeType;  
for(DiffEntry entry : diffEntries){  
    changeType = entry.getChangeType();  
    switch (changeType) {  
        case ADD:  
            updateFiles.add(entry.getNewPath());  
            break;  
        case COPY:  
            updateFiles.add(entry.getNewPath());  
            break;  
        case DELETE:  
            updateFiles.add(entry.getOldPath());  
            break;  
        case MODIFY:  
            updateFiles.add(entry.getOldPath());  
            break;  
        case RENAME:  
            updateFiles.add(entry.getNewPath());  
            break;  
        }  
}  
//将修改类型的文件提交到git仓库中，并返回本次提交的版本号  
AddCommand addCmd = git.add();  
for (String file : updateFiles) {  
    addCmd.addFilepattern(file);  
}  
addCmd.call(); 
CommitCommand commitCmd = git.commit();
```

## 6、提交修改 git commit

```java
Status status = git.status().call();
String commitId = null;
if (!status.isClean()) {
// 执行 git 提交
  git.add().addFilepattern(".").call();
  log.debug("Git add all files success.");
  CommitCommand commit = git.commit();
  // 指定用户提交
  RevCommit call = commit.setMessage(comment).setCommitter("userName-zhimaxingzhe", userName + "@github.com").call();
  commitId = call.getName();
} else {
  log.debug("Git status is clean,nothing to commit.");
  Iterable<RevCommit> call = git.log().setMaxCount(1).call();
  for (RevCommit revCommit : call) {
    commitId = revCommit.getName();
  }
}
```

## 7、推到远程仓库 git push
```java
git.push().setRemote("origin").setCredentialsProvider(provider)
                    .setRefSpecs(new RefSpec(branchName)).call();
```
## 8、切换分支 git checkout
```java
Git git = Git.open(new File("/Users/zhimaxingzhe/test"));
UsernamePasswordCredentialsProvider provider = new UsernamePasswordCredentialsProvider("userName-zhimaxingzhe", password);
// 拉取远程更新        
git.pull().setCredentialsProvider(provider).setRemoteBranchName(branchName).call();
boolean existBranch = false;
//列出所有的分支名称，判断分支是否存在
List<Ref> branchList = git.branchList().setListMode(ListBranchCommand.ListMode.ALL).call();
String branchName = featureInfo.getBranchVersion();
for (Ref ref : branchList) {
    if (("refs/remotes/origin/" + branchName).equals(ref.getName())) {
        existBranch = true;
    }
}
if (!existBranch) {
    log.error("分支{}不存在，请确认", branchName);
    throw new BusinessException(ExceptionEnum.BRANCH_NOT_EXIST.getCode(),
            "分支" + branchName + "不存在，请确认");
}
boolean existLocalBranch = false;
List<Ref> branchList = git.branchList().call();
for (Ref ref : branchList) {
    if (ref.getName().equals("refs/heads/" + branchName)) {
        existLocalBranch = true;
    }
}
if (existLocalBranch) {
    // 存在则切换本地分支
    git.checkout().setName(branchName).call();
} else {
    // 不存在则切换为远程分支
    git.checkout().setCreateBranch(true).setName(branchName)
            .setUpstreamMode(CreateBranchCommand.SetupUpstreamMode.TRACK)
            .setStartPoint("origin/" + branchName).call();
}
```

## 9、回滚到指定版本 git reset
```java
/**
     * 将git仓库内容回滚到指定版本，回退前要考虑版本所处分支和当前仓库分支是否一致，否则回退失败
     *
     * @param gitRoot 仓库目录
     * @param commitId 指定的版本号
     * @return true, 回滚成功, 否则flase
     * @throws IOException
     */
    public boolean rollBackPreRevision(String gitRoot, String commitId) throws IOException {
        // Git实现了AutoCloseable，默认会关闭repository
        try (Git git = Git.open(new File(gitRoot))){
            git.reset().setMode(ResetType.HARD).setRef(commitId).call();
        } catch (GitAPIException e) {
            log.error("回退失败",e);
            throw new BusinessException("回退失败");
        }
        return true;
    }

    /**
     * 将git仓库内容回滚到指定版本的上一个版本，回退前要考虑版本所处分支和当前仓库分支是否一致，否则回退失败
     *
     * @param gitRoot 仓库目录
     * @param commitId 指定的版本号
     * @return true, 回滚成功, 否则flase
     * @throws IOException
     */
    public boolean rollBackPreRevision(String gitRoot, String commitId) throws IOException {
        // Git实现了AutoCloseable，默认会关闭repository
        try (Git git = Git.open(new File(gitRoot))){
            Repository repository = git.getRepository();
            // 获取 commitId 的上一个版本
            RevWalk walk = new RevWalk(repository);
            ObjectId objId = repository.resolve(commitId);
            RevCommit revCommit = walk.parseCommit(objId);
            String preVision = revCommit.getParent(0).getName();
        
            git.reset().setMode(ResetType.HARD).setRef(preVision).call();
        } catch (GitAPIException e) {
            log.error("回退失败",e);
            throw new BusinessException("回退失败");
        }
        repository.close();
        return true;
    }
```
## 10、比较版本差异 git diff
比较当前工作区和和最后一次提交的差异
 ```java
 List<DiffEntry> call = git.diff().call();
 ``` 
 比较两个版本之间的差异
```java
// git diff HEAD HEAD^
ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
AbstractTreeIterator newTreeIter = prepareTreeParser(git.getRepository(), git.getRepository().resolve("HEAD").getName());
AbstractTreeIterator oldTreeIter = prepareTreeParser(git.getRepository(), git.getRepository().resolve("HEAD^").getName());
git.diff()
    .setNewTree(newTreeIter)  //设置源，不设置则默认工作区和历史最新commit版本比较
    .setOldTree(oldTreeIter)
    .setPathFilter(PathFilter.create("README.dm"))  //设置过滤
    .setOutputStream(outputStream) //输出流  用outputStream，后面转成字符串
    .call();
```
如果只是想知道本地工作区有没有可提交的东西，可以使用git status，判断is clean即可。

```java
Status status = git.status().call();
if (!status.isClean()) {
  // do something
}
```
查看源码可以看到status().call()内部也是调用了diff.diff()来获得状态的。

## 11、合并分支 git merge  [sourceBranchName]
```java
Git git = Git.open(new File("/Users/zhimaxingzhe/test"));
Ref refdev = git.checkout().setName(sourceBranchName).call(); //切换源头分支获取源头分支信息
git.checkout().setName(targetBranchName).call();   //切换回被合并分支
// git merge [sourceBranchName]
MergeResult mergeResult = git.merge().include(refdev)  // 合并源头分支到目标分支
        .setCommit(true)           // 设置合并后同时提交
        .setFastForward(MergeCommand.FastForwardMode.NO_FF)// 分支合并策略，--ff代表快速合并， --no-ff代表普通合并
        .setMessage("Merge sourceBranchName into targetBranchName.")     //设置提交信息
        .call();
```

# 三、源码学习
## 1、JGit pull 源码阅读
pull 实际上是 fetch、merge两个操作的组合，那么在JGit 源码中这也是执行了这两个操作，带着这样的预期进入方法中，看到关键步骤：
```java
FetchCommand fetch = new FetchCommand(repo).setRemote(remote)
        .setProgressMonitor(monitor).setTagOpt(tagOption)
        .setRecurseSubmodules(submoduleRecurseMode);
configure(fetch);

fetchRes = fetch.call();
```
```java
MergeCommand merge = new MergeCommand(repo);
MergeResult mergeRes = merge.include(upstreamName, commitToMerge)
        .setProgressMonitor(monitor)
        .setStrategy(strategy)
        .setContentMergeStrategy(contentStrategy)
        .setFastForward(getFastForwardMode()).call();
monitor.update(1);
result = new PullResult(fetchRes, remote, mergeRes);
```
## 2、JGit clone 源码阅读
JGit 是可以不依赖服务器安装 git client 即可对 GIT 进行操作的，是如何做到的呢？带着这个疑问从 JGit 的 clone()方法 源码开始寻找答案。
通常git仓库的目录结构是这样的

![git init](https://zhimaxingzhe.github.io/image/jgit/git_init.png)  
JGit也是创建了这些目录和文件。
clone 的关键代码步骤：
1、创建各种目录和文件

```java
refs.create();
objectDatabase.create();
FileUtils.mkdir(new File(getDirectory(), "branches")); //$NON-NLS-1$
FileUtils.mkdir(new File(getDirectory(), "hooks")); //$NON-NLS-1$
// ......
```
2、执行 fetch

```java
FetchResult result = transport.fetch(monitor,
applyOptions(refSpecs), initialBranch);
```
3、执行 checkout()

```java
DirCache dc = clonedRepo.lockDirCache();
DirCacheCheckout co = new DirCacheCheckout(clonedRepo, dc,commit.getTree());
co.setProgressMonitor(monitor);
co.checkout();
```

## 3、JGit push 源码阅读
git push命令用于将本地分支的更新，推送到远程主机。它的格式与git pull命令相仿。
分支推送顺序的写法是<来源地>:<目的地>，所以git pull是<远程分支>:<本地分支>，而git push是<本地分支>:<远程分支>。
如果省略远程分支名，则表示将本地分支推送与之存在”追踪关系”的远程分支(通常两者同名)，如果该远程分支不存在，则会被新建。



```java
//获取钩子
prePush = Hooks.prePush(local, hookOutRedirect, hookErrRedirect);
// 执行推送
PushProcess pushProcess = new PushProcess(this, toPush, prePush, out);
return pushProcess.execute(monitor);
```
pushProcess.execute(monitor) 方法这里面太复杂了，晚点再学🤦‍♂️。

其他的API还有很多值得研读的，比如 commit 是如何操作工作区和本地仓库的，很多命令带上参数在JGit上能否对应实现，又是如何实现对应操作的，比如 checkout 带上 setCreateBranch是如何实现分支的创建的，带着对git的理解看源码，既可以了解JGit的实现，通过JGit API 中的代码实现以及很多边界条件的考虑，还可以加深对git本身的了解。



还有更多介绍可以查看[JGit 官方主页](https://www.eclipse.org/jgit/) [JGit官方API文档](https://wiki.eclipse.org/jgit/User_Guide)  [JGit Support ](https://www.eclipse.org/jgit/support/)

## 附加信息

### Java 执行 shell 命令
从接口权限、方法权限、启动应用用户拥有哪些命令权限都要控制好，否则就是木马般的存在🤦🏻。
```xml
<!-- Test Dependencies 存在本地信息泄露漏洞，junit 版本要在4.13.1版本以上 https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-15250 -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-exec</artifactId>
    <version>1.3</version>
</dependency>

```
```java
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.exec.CommandLine;
import org.apache.commons.exec.DefaultExecutor;
import org.apache.commons.exec.ExecuteWatchdog;
import org.apache.commons.exec.PumpStreamHandler;

@Slf4j
public class Test {

    private String executeCommand(String command, String params) {
        String line = String.format("%s %s", command, params);
        return execute(line);
    }

    private String executeCommand(String command) {
        return execute(command);
    }

    private String execute(String line) {
        CommandLine commandLine = CommandLine.parse(line);
        DefaultExecutor defaultExecutor = new DefaultExecutor();
        defaultExecutor.setExitValues((int[]) null);
        ExecuteWatchdog watchdog = new ExecuteWatchdog(60000L);
        defaultExecutor.setWatchdog(watchdog);
        ByteArrayOutputStream os = new ByteArrayOutputStream();
        ByteArrayOutputStream es = new ByteArrayOutputStream();
        PumpStreamHandler streamHandler = new PumpStreamHandler(os, es);
        defaultExecutor.setStreamHandler(streamHandler);
        String out = "";
        String err = "";

        try {
            defaultExecutor.execute(commandLine);
            out = os.toString(StandardCharsets.UTF_8.name());
            err = es.toString(StandardCharsets.UTF_8.name());
        } catch (IOException var16) {
            log.warn("defaultExecutor exe command IO error:", var16);

        }

        if (err != null && err.length() > 0) {
            log.error("execute command {} error:{}", line, err);
            throw new BusinessException("execute command " + line + "\n error " + err);
        } else {
            return out;
        }
    }
}
``` 



参考材料：  
[jgit-cookbook](https://github.com/centic9/jgit-cookbook) 这个仓库种包含了操作 Jgit 的丰富样例，很赞。