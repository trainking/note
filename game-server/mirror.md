# Mirror

Mirror是一个封装高层级的Unity的网络库。但不单单是网络库，在实际开发中，可以依赖它实现状态传输的服务器实现。在开发3D游戏中有效，客户端和服务端应用的是相同的代码，大大减少了开发成本，同时避免了浮点数物理运算导致不一致问题。

> 建议使用Unity 2020或者2021 LTS版本。

## 安装

从Unity的资源商店，搜索`Mirror`安装。

## 基础概念

### NetworkManager

**NetworkManager**组件，是Mirror的核心组件，它定义网络抽象定义。Mirror封装高层，开发无须关心起底层实现。只需要在场景中，使用空对象创建一个NetworkManager的预制件，此预制件引用了NetworkManager组件。

**NetworkManager**有以下比较重要属性需要关注：

* `Offline Scene`：设置离线时场景
* `Online Scene`：设置在线时场景
* `Transport`：协议脚本引用，例如`KCP Script`
* `Network Address`: 连接地址
* `Max Connections`: 最大连接数
* `Player Prefab`：玩家预制件
* `Auto Create Player`：是否自动创建玩家
* `Player Spawn Method`: 玩家出生点的方法，Random和Round Robin

使用`NetworkManager`传输网络对象的GameObject，需要将实现的`Behaviour`改为`NetworkBehaviour`:

```c#
public class PlayerScript : NetworkBehaviour
{
    ...
}
```

**使用注解**

**NetworkManager**实现了一套使用注解定义网络传输对象的机制，主要如下:

|注解|使用对象|说明|额外参数|
|:--|:--|:--|:--|
|SyncVar|属性|属性变化，会同步到其他客户端|hook: 值变化的观察者|
|Command|方法|同步指令，在客户端调用在服务器运行||
|ClientRpc|方法|同步调用，在服务器调用在客户端运行|

> 详情参考[远程操作](###远程操作)

### Kcp Transport

**Kcp Transport**是一个脚本，定义了KCP协议的实现。**NetworkManager**本身是网络对象的高层实现，所以需要通过传入Transport对象，实现具体的协议传输。

### 远程操作

Mirror提供了两种远程过程调用的方式，`Command`和`ClientRpc`，并提供了各自的注解。这两种的区别在于：

* Command：从客户端调用，并在服务器上运行
* ClientRpc: 在服务器上调用，并在客户端上运行

![示意图](https://3359251531-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MGmQrf2z6FL0ZpExPAn%2F-MH_qTfPEnSOCoRntjcs%2F-MH_quqZtwY246Y5QMcA%2Fimage.png?alt=media&token=01f744f7-c7bb-4462-86dc-9844a0ed5cbe)

#### Command

Command是客户端上的玩家对象，像服务器上的玩家对象发送指令。安全起见，默认情况下，玩家只能给自己的对象镜像（服务器上）发送指令，玩家无法操作其他玩家。可以使用`[Command(requiresAuthority = false)]`绕过检查。

Command注解的函数，必须使用`Cmd`开头的函数名。当客户端调用时，它将在服务器端运行。允许任何类型的值，作为函数的参数传递:

```C#
    [Command]
    private void CmdSetupPlayer(string nameValue, Color colorValue)
    {
        playerName = nameValue;
        playerColor = colorValue;
    }
```

#### ClientRpc

ClientRpc调用是从服务器上的对象发送客户端对象。可以从任何服务器对象发起，因为服务器拥有所有权限，所以不存在不能调用的问题。函数怎就该`[ClientRpc]`注解，即可成为一个`Rpc Call`。同时，函数名必须使用`Rpc`开头：

```C#
    public void TakeDamage(int amount)
    {
        if (!isServer) return;

        health -= amount;
        RpcDamage(amount);
    }

    [ClientRpc]
    public void RpcDamage(int amount)
    {
        Debug.Log("Took damage:" + amount);
    }
```

## 组件使用

### Network Transform

使用此组件，可以同步对象的Transform，在其`Sync Setting`中，有两种模式：`Client To Server`和`Server To Client`。

**Client To Server**

客户端计算`transform`结果，同步到服务器。适合用于同步玩家的位置，一个是要从客户端获取操作，二是，将计算压力留给客户端。

**Server To Client**

服务端计算`transform`结果，同步给各个客户端。适合同步被动物体，例如足球游戏中的足球，需要注意的是，此模式下有两个选项：Observes和Owner。Observes是观察者模式，对数据没有足够精确，会有穿模的效果。Owner则是以物体本身为准，精确，不会有穿模，但是计算量相对大。

### Network Room Manager

Network Room Manager 是一个方便的网络房间管理器，它可以帮助我们快速实现多人游戏中的房间管理功能。