# 治理手册

这一节主要讲解如何治理权限合约。如果您是业务方，无需阅读接下来的内容；如果您是治理方，想了解如何部署权限合约、配置权限规则，则需要继续阅读本节。 

## 1. 关键概念

### 1.1.1 业务合约与权限合约

权限治理涉及的合约分为两类：业务合约、权限合约。

- 业务合约是用户自己开发的合约，不属于本组件。业务合约通过访问权限合约的canCallFunctoin函数来拦截非法调用。
- 权限合约是本组件提供的AuthManager合约，用于配置哪些账户可以访问哪些合约的哪些函数。可以治理多个业务合的权限。

### 1.1.2 组
组定义了哪些账户可以访问哪些函数。组包含的信息如下：

- 账户列表
- 模式。包含两个可选择的模式：
    * 黑名单模式
    * 白名单模式
- 函数列表

通过上面的配置，即可确定哪些账户可以访问哪些函数:
- 配置了一个黑名单组，意味着仅组内账户不能访问这些函数
- 配置了一个白名单组，意味着仅组内账户可以访问这些函数

### 1.1.3 治理模式
权限治理合约有两种模式：治理员模式、委员会模式

- 治理员模式下，由单一治理员修改组配置； 还可以转让治理员权限。
- 委员会模式下，所有操作均通过投票进行。委员会成员可以修改组配置。还可以修改委员会列表、投票规则等。

一个权限治理合约的模式在部署时即确定，一旦确定一种模式，就无法更改。

### 1.1.4 投票模式
投票包含两种规则：多签模式和阈值权重模式。

- 多签模式下，当投票数达到最小签名数时，投票即通过。
- 阈值权重模式下，每个委员均配有对应的权重，当已投票的总权重达到最小阈值时，投票即通过。

多签是阈值权重模式下的一个特例，即所有委员会的权重为1。


## 2. AuthManager合约接口列表
- 合约部署

合约部署时需要决定是治理员模式还是治理委员会模式。以下为治理员模式下权限配置接口：
- createGroup 创建组
- addAccountToGroup 将账户添加到组
- addFunctionToGroup 将合约函数关联到组
- removeAccountFromGroup 将账户从组内移除
- removeFunctionFromGroup 将合约函数与组的关联取消

以下为治理委员会模式下的权限配置接口：
- requestCreateGroup 请求创建组
- requestAddAccountToGroup 请求将账户添加到组
- requestAddFunctionToGroup 请求将合约函数关联到组
- requestRemoveAccountFromGroup 请求将账户从组内移除
- requestRemoveFunctionFromGroup 请求将合约函数与组的关联取消
- approveSingle 投票请求
- deleteSingle 删除请求
- getRequestSingle 查看请求
- executeCreateGroup 创建组
- executeAddAccountToGroup 执行将账户添加到组
- executeAddFunctionToGroup 执行将合约函数关联到组
- executeRemoveAccountFromGroup 执行将账户从组内移除
- executeRemoveFunctionFromGroup 执行将合约函数与组的关联取消

以下为通用的查询接口
- containsAccount
- containsFunction
- getGroup
- canCallFunction

以下为治理员模式下的治理接口：
- transferAdminAuth
- isAdmin

以下是治理委员会模式下的治理接口：
- requestSetThreshold
- requestResetGovernors
- executeSetThreshold
- executeResetGovernors
- requestAddGovernor
- deleteAddGovernorReq
- approveAddGovernorReq
- getGovernorsToAdd
- executeAddGovernorReq
- requestRemoveGovernor
- deleteRemoveGovernorReq
- approveRemoveGovernorReq
- getGovernorsToRemove
- executeRemoveGovernorReq

### 2.1. 合约部署

说明：部署权限治理合约AuthManager。

参数说明：
- uint mode：1-治理员模式，2-治理委员会模式。如果选择治理员模式，则当前账户为治理员。
- address[] accounts: 初始的治理委员会列表。仅mode为2时有效，如果mode为1，则本参数传空数组即可。
- uint16[] weights: 初始治理委员会权重。仅mode为2时有效，如果mode为1，则本参数传空数组即可。
- uint16 threshold: 投票有效的最小权重。仅mode为2时有效，如果mode为1，则本参数传0即可。

示例1：部署治理员模式下的权限治理合约。后面三个参数如本例一样传空即可。

```
deploy AuthManager 1 [,] [,] 0
```

示例2：部署委员会模式的权限治理合约，并且采用多签投票方式。比如委员会包含三个地址，分别为"0x1","0x2","0x3"，要求至少3票投票才能通过，则部署方式为：

```
deploy AuthManager 2 ["0x1", "0x2", "0x3"] [1,1,1] 3 
```

这里[1,1,1]表示每个委员权重为1。

示例3：部署委员会方式的权限治理合约，并且采用基于阈值投票方式。比如委员会包含三个地址，分别为"0x1","0x2","0x3"，权重分别为1，2，3。要求总权重为4，投票才算有效。则部署方式为：

```
deploy AuthManager 2 ["0x1","0x2", "0x3"] [1,2,3] 4
```

### 2.2. createGroup

说明：创建一个组。

调用要求：当前调用者为治理员。该组必须存在。

参数说明：
- string group: 组名
- uint8 mode：组是黑名单还是白名单。1-白名单，2-黑名单

### 2.3. addAccountToGroup

说明：将账户地址拉入到组中。

调用要求：要求当前调用者为治理员。要求组已被创建，且账户尚未在这个组中。

参数说明：
- address account: 账户地址
- string group: 组名

### 2.4. addFunctionToGroup

说明：将某合约的某函数关联到组中。调用后，若当前组为白名单组，则仅有组内账户可以访问该函数；若当前组为黑名单组，则仅有组内账户不允许访问该函数。

调用要求：当前调用者为治理员。要求组已被创建，且函数未被关联到任何组中（无论是本组，还是其他组）

参数说明：
- address contractAddr: 业务合约地址
- string func: 业务合约中待关联函数的签名字符串，如"add(uint256,uint256)","hello()"等。
- string group: 组名

### 2.5. removeAccountFromGroup

说明：将账户地址从组中移除。

调用要求：当前调用者为治理员。要求组已被创建，且账户在这个组中。

参数说明：
- address account: 账户地址
- string group: 组名

### 2.6. removeFunctionFromGroup

说明：取消某合约某函数与组的关联。

调用要求：当前调用者为治理员。要求组已被创建，且函数已被关联到组。

参数说明：
- address contractAddr: 业务合约地址
- string func: 业务合约中待关联函数的签名字符串，如"add(uint256,uint256)","hello()"等。
- string group: 组名

### 2.7. requestCreateGroup

说明：请求创建一个组。只能同时存在一个同类请求。参数会被缓存，直到请求被执行或删除。

调用要求：当前调用者为治理委员会成员；不存在其他未关闭的同类请求。

参数说明：
- string group: 组名
- uint8 mode：组是黑名单还是白名单。1-白名单，2-黑名单

### 2.8. requestAddAccountToGroup

说明：请求将账户添加到组。只能同时存在一个同类请求。参数会被缓存，直到请求被执行或删除。

调用要求：当前调用者为治理委员会成员；不存在其他未关闭的同类请求。

参数说明：
- address account: 账户地址
- string group: 组名

### 2.9. requestAddFunctionToGroup

说明：请求将函数关联到组。只能同时存在一个同类请求。参数会被缓存，直到请求被执行或删除。

调用要求：当前调用者为治理委员会成员；不存在其他未关闭的同类请求。

参数说明：
- address contractAddr: 业务合约地址
- string func: 业务合约中待关联函数的签名字符串，如"add(uint256,uint256)","hello()"等。
- string group: 组名

### 2.10. requestRemoveAccountFromGrop

说明：请求将账户地址从组中移除。只能同时存在一个同类请求。参数会被缓存，直到请求被执行或删除。

调用要求：当前调用者为治理委员会成员；不存在其他未关闭的同类请求。

参数说明：
- address account: 账户地址
- string group: 组名

### 2.11. requestRemoveFunctionFromGroup

说明：请求取消某合约某函数与组的关联。只能同时存在一个同类请求。参数会被缓存，直到请求被执行或删除。

调用要求：当前调用者为治理委员会成员；不存在其他未关闭的同类请求。

参数说明：
- address contractAddr: 业务合约地址
- string func: 业务合约中待关联函数的签名字符串，如"add(uint256,uint256)","hello()"等。
- string group: 组名

### 2.12. approveSingle

说明：对某个提案进行投票。

调用要求：当前调用者为治理委员会成员；投票还未关闭。

参数说明：
- uint8 txType：请求类型。见《请求类型列表》一节。

### 2.13. deleteSingle

说明：删除某个提案。

调用要求：当前调用者为治理委员会成员；投票还未关闭。

参数说明：
- uint8 txType：请求类型。见《请求类型列表》一节。

### 2.14. getRequestSingle

说明：获取某个未关闭的投票信息。

调用要求：投票还未关闭。

参数说明：
- uint8 txType：请求类型。见《请求类型列表》一节。

返回值：
- uint256：请求id（可以忽略）
- address：请求的发起地址
- uint256：请求的投票总阈值
- address：（可以忽略）
- uint256：目前投票的总权重
- uint8：请求类型。见《请求类型列表》一节。
- uint8：（可以忽略）


### 2.15. executeCreateGroup

说明：执行创建组。

调用要求：当前调用者为治理委员会成员；请求已被投票通过。

### 2.16. executeAddAccountToGroup

说明：执行将账户添加到组的请求。

调用要求：当前调用者为治理委员会成员；请求已被投票通过。

### 2.17. executeAddFunctionToGroup

说明：执行将业务函数关联到组的请求。

调用要求：当前调用者为治理委员会成员；请求已被投票通过。

### 2.18. executeRemoveAccountFromGroup

说明：执行将账户从组中移除的请求。

调用要求：当前调用者为治理委员会成员；请求已被投票通过。

### 2.19. executeRemoveFunctionFromGroup

说明：执行取消业务函数和组关联的请求。

调用要求：当前调用者为治理委员会成员；请求已被投票通过。

### 2.20. containsAccount

说明：查询某一账户是否位于某一组中。

调用要求：无

参数说明：
- string group:组名
- address account：账户

### 2.21. containsFunction

说明：查询某一合约函数是否已关联到组。

调用要求：无

参数说明：
- string group：组名
- address contractAddr：业务合约地址
- string func:业务合约中待关联函数的签名字符串，如"add(uint256,uint256)","hello()"等。

### 2.22. getGroup

说明：查询某一个组的信息

调用要求：无

参数说明：
- string group:组名

返回值说明：
- uint8：组的模式。1-白名单，2-黑名单
- uint256：组包含的账户数目
- uint256：组关联的函数数目

### 2.23. canCallFunction

说明：查询某一账户是否有权限调用某一合约函数

调用要求：无

参数说明：
- address contractAddr：业务合约地址
- bytes4 sig：业务函数的签名字节，由sha3(函数签名字符串)的前4字节得来。和msg.sig一致。
- address caller：调用者

### 2.24. transferAdminAuth

说明：转移治理员权限给另一个账户。

调用要求：当前调用者为治理员。

参数说明：
- address newAdminAddr：新治理员的账户地址。

### 2.25. isAdmin

说明：判断当前调用者是否为合约治理员。

调用要求：无



### 2.26. requestSetThreshold

说明：请求重设投票权重阈值。只能同时存在一个同类请求。参数会被缓存，直到请求被执行或删除。

调用要求：当前调用者为治理委员会成员；不存在其他未关闭的同类请求。

参数说明：
- uint16 newThreshold：新的权重阈值


### 2.27. requestResetGovernors

说明：请求重设治理委员会列表与权重。只能同时存在一个同类请求。参数会被缓存，直到请求被执行或删除。

调用要求：当前调用者为治理委员会成员；不存在其他未关闭的同类请求。

参数说明：
- address[] governors: 新的治理委员会名单
- uint16[] weights: 新的治理委员会对应权重

 ### 2.28. executeSetThreshold

说明：执行重设阈值请求

调用要求：当前调用者为治理委员会成员；请求已被投票通过。

### 2.29. executeResetGovernAccounts

说明：执行重设治理委员会列表请求

调用要求：当前调用者为治理委员会成员；请求已被投票通过。

### 2.30. requestAddGovernor

说明：请求向治理委员会中添加一个成员

调用要求：当前调用者为治理委员会成员；待添加账户还未没有过相关添加请求

参数说明：
- address account: 待添加成员

### 2.31. deleteAddGovernorReq

说明：删除添加治理委员会成员请求

调用要求：当前调用者为治理委员会成员；待添加账户已经生成了添加请求

参数说明：
- address account: 待添加成员

### 2.32. approveAddGovernorReq

说明：同意添加治理委员会

调用要求：当前调用者为治理委员会成员；待添加账户已经生成了添加请求

参数说明：
- address account: 待添加成员

### 2.33. getGovernorsToAdd

说明：取得所有待添加成员

返回值：
- address[]: 所有待添加的成员名单

### 2.34. executeAddGovernorReq

说明：执行请求

调用要求：当前调用者为治理委员会成员；待添加账户已经生成了添加请求

### 2.35. requestRemoveGovernor

说明：请求从治理委员会中删除一个成员

调用要求：当前调用者为治理委员会成员；待移除账户还未没有过相关移除请求

参数说明：
- address account: 待移除成员

### 2.36. deleteRemoveGovernorReq

说明：删除删除治理委员会成员请求

调用要求：当前调用者为治理委员会成员；待移除账户已经生成了移除请求

参数说明：
- address account: 待移除成员

### 2.37. approveRemoveGovernorReq

说明：同意删除治理委员会成员

调用要求：当前调用者为治理委员会成员；待移除账户已经生成了移除请求

参数说明：
- address account: 待移除成员

### 2.38. getGovernorsToRemove

说明：取得所有待删除成员

返回值：
- address[]: 所有待移除的成员名单

### 2.39. executeAddGovernorReq

说明：执行请求

调用要求：当前调用者为治理委员会成员；待添加账户已经生成了添加请求

## 3. 常量表

### 3.1. 组相关
- 白名单组：1
- 黑名单组：2

### 3.2. 请求类型相关

- 重设投票阈值：1
- 重设治理委员会列表：2
- 创建组：3
- 添加账户到组：4
- 关联函数到组：5
- 组内移除函数：6
- 取消函数和组关联：7

## 4. 集成javaSdk
在前面示例中，我们以控制台来操作部署、操作权限合约。除了控制台外，本组件也支持通过java代码来执行这些操作。推荐使用集成SDK的方式来进行权限治理，这样可以在没有控制台的情况下进行权限合约调用。

### 4.1. 源码下载

先前章节中，已经通过git下载了源码：
```
git clone git@github.com:WeBankBlockchain/Gov-Auth.git 
cd auth-manager
```

这个源码包含：
- 权限治理合约
- 权限治理的java sdk

### 4.2. 编译

方式一：如果服务器已安装Gradle

```
gradle build -x test
```

方式二：如果服务器未安装Gradle，则使用gradlew编译。Linux环境：gradlew。Windows环境：gradlew.bat。
以Linux环境为例：
```
chmod +x ./gradlew && ./gradlew build -x test
```

编译过后，得到jar包：dist/auth-manager.jar。

### 4.3 集成
在编译好该jar包后，将它引入到一个示例项目中进行调用，以操作权限合约。

#### 4.3.1. 新建springboot项目

可以通过 https://start.spring.io/ 等方式来新建一个springboot项目。

#### 4.3.2. 引入auth-manager sdk

前文中，编译auth-manager项目后得到了dist\auth-manager.jar。可以将auth-manager.jar导入到自己的项目中，例如拷贝到libs目录下，然后进行依赖配置，再对自己的项目进行编译。推荐gradle配置如下，

```
repositories {
    maven { url "http://maven.aliyun.com/nexus/content/groups/public/"}
    maven { url "https://oss.sonatype.org/content/repositories/snapshots" }
    maven { url "https://dl.bintray.com/ethereum/maven/" }
    mavenLocal()
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    testCompile('org.springframework.boot:spring-boot-starter-test') {
        exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
        //exclude group: 'junit', module: 'junit'
    }
    compile ('org.fisco-bcos:web3sdk:2.2.2')
    compile fileTree(dir:'libs',include:['*.jar'])
}
```

#### 4.3.3 配置

从控制台拷贝如下几个文件到业务工程的main/resources目录下：

- conf/ca.crt, conf/node.key, conf/node.crt

拷贝后，业务系统main/resources目录应为如下结构：

```
resources
 - ca.crt
 - node.crt
 - node.key
 - application.properties
```

配置application.properties：

```
### chain
# 机构名
system.orgId=jigou
# 节点连接字符串
system.nodeStr=[节点连接字符串]
# 群组id
system.groupId=1
# 加密方式，0-ECC， 1-国密
system.encryptType=0

## 私钥，请更换为对应16进制私钥，不要以0x前缀
system.privateKey=
```
#### 4.3.4 使用方式示例

本例在单元测试中完成调用示例。首先新建DemoTest类，并通过Spring自动注入得到AuthManagerFactory:

```
@SpringBootTest
public class DemoTest {

    @Autowired
    private AuthManagerFactory factory;

    @Test
    public void test() throws Exception{
        //TODO     
    }
}
```

接下来设置黑名单，将一个账户拉黑，使得它无法调用hello函数。

```
@Test
public void demo() throws Exception{
    //创建权限合约
    AuthManager authManager = factory.createAdmin();
    //创建治理员调用接口
    AuthByAdminService authByAdminService = new AuthByAdminService(authManager);
    //创建组
    String group = "badGroup";
    authByAdminService.createGroup(group, AuthConstants.ACL_BLACKLIST_MODE);
    String blackAccount = "[待拉黑账户]";
    authByAdminService.addAccountToGroup(blackAccount, group);
    //配置组权限
    String bizContractAddress = "[HelloWorld合约地址]";
    String function = "[hello函数签名]";
    authByAdminService.addFunctionToGroup(bizContractAddress, function, group);
    //验证
    boolean canCall = authByAdminService.canCallFunction(bizContractAddress, function, blackAccount);
    Assert.assertFalse(canCall);
}
```

## 5. 常见问题

### 5.1. jvm崩溃

现象：

```

#
# A fatal error has been detected by the Java Runtime Environment:
#
#  Internal Error (sharedRuntime.cpp:834), pid=17781, tid=140031174805248
#  fatal error: exception happened outside interpreter, nmethods and vtable stubs at pc 0x00007f5c1d05406f
#
# JRE version: Java(TM) SE Runtime Environment (8.0_45-b14) (build 1.8.0_45-b14)
# Java VM: Java HotSpot(TM) 64-Bit Server VM (25.45-b02 mixed mode linux-amd64 compressed oops)
# Failed to write core dump. Core dumps have been disabled. To enable core dumping, try "ulimit -c unlimited" before starting Java again
#
# If you would like to submit a bug report, please visit:
#   http://bugreport.java.com/bugreport/crash.jsp
#

---------------  T H R E A D  ---------------

Current thread (0x00007f5bac160800):  JavaThread "nioEventLoopGroup-3-12" [_thread_in_Java, id=18017, stack(0x00007f5b8c5e8000,0x00007f5b8c6e9000)]

Stack: [0x00007f5b8c5e8000,0x00007f5b8c6e9000],  sp=0x00007f5b8c6e6150,  free space=1016k
Native frames: (J=compiled Java code, j=interpreted, Vv=VM code, C=native code)
V  [libjvm.so+0xaac99a]  VMError::report_and_die()+0x2ba
V  [libjvm.so+0x4f2de9]  report_fatal(char const*, int, char const*)+0x59
V  [libjvm.so+0x9ab5ba]  SharedRuntime::continuation_for_implicit_exception(JavaThread*, unsigned char*, SharedRuntime::ImplicitExceptionKind)+0x33a
V  [libjvm.so+0x914f1a]  JVM_handle_linux_signal+0x48a
V  [libjvm.so+0x90b493]  signalHandler(int, siginfo*, void*)+0x43
C  [libpthread.so.0+0xf100]
J 10415 C2 com.sun.crypto.provider.GCTR.doFinal([BII[BI)I (130 bytes) @ 0x00007f5c1e10c096 [0x00007f5c1e10be40+0x256]

```

这是jdk的bug，需要将jdk版本升级到jdk8u51版本。

### 5.2. 函数签名是什么

函数签名是函数的标识符，格式为“函数名(参数类型列表)”。例如，有下面这个solidity函数：
```
function add(uint256 a, uint256 b) public pure returns(uint256){}
```

那么它的函数签名为：
```
add(uint256,uint256)
```