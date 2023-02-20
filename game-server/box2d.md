# Box2D

**Box2D** 是一个开源（基于MIT协议）的2D物理引擎，详情访问[box2d.org](https://box2d.org/)。使用可移植的C++写成，引擎定义的类型，都是以b2前缀开头。

## 安装

### C++版本

#### 类Linux

类Liux系统，只需要将代码clone下来，直接执行`build.sh`脚本即可。可能会报确实`cmake`工具，安装即可。

#### windows

windows下，使用提供的`build.bat`会报一个`缺失sln`的错误。也是缺失`cmake`工具导致，安装好`cmake`工具之后，同时要有Visual Studio工具，便可以编译成功。

### C#版本

C#版本使用：

```url
https://github.com/Zonciu/Box2DSharp
```

## 基础知识

物理模拟在计算机图形模拟中，无处不在。游戏开发中，物理模拟是为了反馈给玩家更加真实的效果。或者说，至少是玩家觉得像一种真实的反馈。因此，对基本物理知识质量，力，扭矩等等这些了解是必须的。

### 世界（Wrold）

一个物理世界就是物体，形状和约束相互作用的集合。也可以看作，一个世界就是一个容器。将创建的2D对象，都是需要放到一个世界里面。或者说，依赖世界来创建。**Box2D**可以创建多个世界，但这通常是不必要的。

### 刚体（Body）

`Body`是世界存在的基础对象，后续介绍的各种约束和载具，形状等等，都是一个Strcut结构体。他们传递给Body，应用到World中。`Body`通过`BodyType`来定义不同类型的物体，有三种类型：

* StaticBody: 0质量，0速度，可以被手动移动
* KinematicBody：0质量，用户指定的速度，可以被系统移动。可以理解为一个有无穷大质量的的物体，Box2D中的KinematicBody的碰撞，只发生在KinematicBody与DynamicBody之间，不会与StaticBody和其他KinematicBody相互碰撞
* DynamicBody：正整数的质量，速度由力量决定，可以被系统移动

> `KinematicBody`与`DynamicBody`之间的区别在于，`KinematicBody`并没有物理特性，即它的物理效果完全由碰撞检测之后，由我们自定义代码指定；而`DynamicBody`则会根据物理参数设定（质量，弹性系数，摩擦力等等）产生相应的物理效果

### 形状（Shape）

严格附于物体之上的2D碰撞集合结构。形状具有摩擦（friction）和恢复（restitution）的材料兴致。

* EdgeShape：边形状，可以理解为一堵墙
* CircleShape: 圆形
* PolygonShape：多边形

### 载具（Fixture）

载具将形状绑定到物体之上，并具有一定的材质属性，比如密度(density), 摩擦(friction)和恢复(restitution)。可以看作**包围盒**的实现。

### 约束（Constraint）

约束是消除物体自由度的物理连接，在2D世界中，物体有3个自由度（水平，垂直，旋转）。如果我们把一个物体钉在墙上(像钟摆那样), 那就把它约束到了墙上。这个时候,此物体就只能绕着钉子旋转, 所以这个约束消除了它2个自由度。

#### 接触约束（Contact Constraint）

一种特殊的约束，设计的目的是为了防止刚体被穿透，也用于模拟摩擦和恢复。接触约束不用主动创建，她们会自动被Box2d生成。

### 关节（Joint）

关节就是中约束，将两个或多个Body固定在一起。Box2D支持不同的关节类型:转动(revolute),棱柱(prismatic),距离(distance)等。一些关节可以有限制(limits)和马达(motors)。

#### 关节限制（Joint Limit）

关节限制限定了一个关节的运动范围。例如，人类的胳膊只能在某一个角度范围内运动。

#### 关节马达（Joint Motor）

根据关节的自由度，关节马达可以驱动关节连接的物体。可以使用一个马达来驱动一个肘的旋转。

### AABB



### 物理知识

#### 向量和坐标系

**Box2D**是二维的物理引擎，所以它采用的是笛卡尔坐标系， 以左顶点为原点`(0,0)`, 以米为单位。**向量**（vector）是一个从数学物理学到计算机图形学等等中，基础的概念。指同时一个具有大小和方向，且满足平行四边形法则的集合对象。在计算机图形学中，向量的定义与物理学工程学中一致，也被称之为矢量或欧几里得向量，如运动学中的位移，速度，加速度，力学中的力，力矩等等，都可以用向量来描述。

Box2D中，使用了`Vector2`结构体表达向量，它有两个构造函数：

```C#
public Vector2(float value);
public Vector2(float x, float y);
```

#### 质量

质量（Mass）是物体的**惯性**相关的重要参数，Box2D的Body对象，都可以通过`MassData`设置物体的质量参数：

```c#
    public struct MassData
    {
        /// 重量大小，使用kg为单位
        public float Mass;

        /// 质心相对形状远点的位置，例如圆形就是(0,0)
        public Vector2 Center;

        /// 形状关于局部原点的转动惯量
        public float RotationInertia;
    }
```

#### 密度

密度（Density）是物体形状大小平均分配到物体的密度，决定了**接触点**可以触发的碰撞力。Box2D使用一个float类型，接受物体的密度。密度是由，质量除以面积平方得出。

```
Density = kg/m^2
```

#### 角速度

角速度（AngularVelocity）是用来描述物体转动或一质点绕另一质点转动的快慢和转动方向的物理量。简单来说，就是决定物体的旋转的快慢。Box2D中，使用`m/s`为单位。

#### 线性速度

线性速度（LinearVelocity）物体运动的速度。表现为，物体在世界中的运动速度，通过一个`Vector2`接口来确定方向，然后乘以速度倍数，得到速度的值。Box2D中，使用`m/s`为单位。

#### 作用力

作用力（Force）是改变物体运动状态和形变的根本原因。Box2D的施加力的过程取药确定三个参数：

* 力的大小：一个`Vector2`向量
* 施力的点：施加力在物体的哪个点
* Awake状态：施力后是否唤醒物体

#### 摩擦力

摩擦力（Friction）是两个物体相交事，相互之间产生的摩擦力。Box2D可以给物体添加一个摩擦系数，大小在`0~1f`之间的浮点数。

#### 恢复（弹性）

恢复（Restitution）是物体相撞，产生弹力的重要系数。Box2D可以给物体添加一个恢复系数，大小在`0~1f`之间的浮点数。同时， 提供了另一个恢复速度阈值（RestitutionThreshold），通常以 m/s 为单位。超过此速度的碰撞会应用恢复原状（会反弹）。

#### 阻尼

阻尼（Damping）是指摇荡系统或振动系统受到阻滞使能量随时间而耗散的物理现象。例如，给物体施加一个力，物体运动一定距离后，停止的过程，就是由物体的阻尼系数决定。

Box2D有两个物体阻尼，分为**线性阻尼（LinearDamping）**和**角阻尼（AngularDamping）**，分别影响线性速度和角速度。

#### 扭矩

扭矩（Torque）物理中特殊的力矩，等于力和力臂的乘积。是使物体发生转动的一种特殊的力矩，单位是牛米N·m。**扭矩与转速成反比关系**。

## 快速开始

### 1. 创建一个世界

**Box2D**的程序都是基于一个`World`对象开始。**这个对象是管理内存，对象和模拟的物理中心**。创建一个`World`对象，需要两步：

```C#

var world = new World(new Vector2(0, -10));
```

### 2. 创建一个盒子（地图）

2D游戏的地图，通常是一个盒子。这里便需要用到`EdgeShape`这种特殊的形状：

```c#
            Body ground;
            {
                var bd = new BodyDef();
                bd.Position.Set(0.0f, 20.0f);
                ground = World.CreateBody(bd);

                var shape = new EdgeShape();
                var sd = new FixtureDef();
                sd.Shape = shape;
                sd.Density = 0.0f;
                sd.Restitution = restitution;
                sd.Friction = 1.0f;

                // 左边
                shape.SetTwoSided(new Vector2(-20.0f, -20.0f), new Vector2(-20.0f, 20.0f));
                ground.CreateFixture(sd);

                // 右边
                shape.SetTwoSided(new Vector2(20.0f, -20.0f), new Vector2(20.0f, 20.0f));
                ground.CreateFixture(sd);

                // 上边
                shape.SetTwoSided(new Vector2(-20.0f, 20.0f), new Vector2(20.0f, 20.0f));
                ground.CreateFixture(sd);

                // 下边
                shape.SetTwoSided(new Vector2(-20.0f, -20.0f), new Vector2(20.0f, -20.0f));
                ground.CreateFixture(sd);
            }
```

> 形状一般不赋予物理参数，而是通过建立一个载具来为其添加物理参数。

### 3. 创建物体

创建一个物体时，我们需要确定物体的形状，然后再添加载具，为它添加物理参数：

```C#
            {
                var shape = new CircleShape();
                shape.Radius = 1.0f;

                var fd = new FixtureDef();
                fd.Shape = shape;
                fd.Density = 1.0f;

                var bd = new BodyDef();
                bd.BodyType = BodyType.DynamicBody;
                bd.Position.Set(-10.0f + 3.0f * 3, 20.0f);

                bodyB = World.CreateBody(bd);
                bodyB.UserData = 2;
                fd.Friction = 1.0f;
                fd.Restitution = 0.0f;
                bodyB.CreateFixture(fd);
                bodyB.SetAngularVelocity(0.0f);
                //bodyB.SetLinearVelocity(new Vector2(1.0f, 0.0f));
            }
```

### 4. 碰撞检测

Box2D的碰撞检测，是通过给`World`增加一个`IContactListener`接触监听器：

```C#
World.SetContactListener(new PfContactListener());
```

需要实现以下方法：

```C#
    public class PfContactListener : IContactListener
    {
        /// <summary>
        /// 开始接触
        /// </summary>
        /// <param name="contact"></param>
        /// <exception cref="NotImplementedException"></exception>
        public void BeginContact(Contact contact)
        {
            // 玩家相撞
            if (contact.FixtureA.Body.UserData != null && contact.FixtureB.Body.UserData != null)
            {
                if ((int)contact.FixtureB.Body.UserData == 2)
                {
                    contact.FixtureB.Body.SetLinearVelocity(new Vector2(0.5f, 0.5f) * 2);
                    contact.FixtureB.Body.UserData = 3;
                }
            } 
        }

        /// <summary>
        /// 接触结束
        /// </summary>
        /// <param name="contact"></param>
        /// <exception cref="NotImplementedException"></exception>
        public void EndContact(Contact contact)
        {
        }

        // 求解器执行之后
        public void PostSolve(Contact contact, in ContactImpulse impulse)
        {
        }

        // 就解器执行之前
        public void PreSolve(Contact contact, in Manifold oldManifold)
        {
        }
    }
```

#### 5. 开始世界模拟

Box2D使用积分器的计算方法，积分器是模拟离散时间点的物理方程。即假设，游戏的整个过程，就像我们一页页翻看连环画一样。Box2D必须选择一个时间步长，物理引擎通常喜欢60H（1/60，即一秒60帧）这样的事件步长，但是在服务器跑这样的逻辑时，不能使用这么大的步长。

除了积分器之外，Box2D还使用了一个更大的特性，叫做约束结算器（solver）。Solver一次解决模拟中的所有约束，一次一个。在Solver阶段，有两个阶段：**速度阶段**和**位置阶段**。

在速度阶段，Solver算Body移动所需要的脉冲。在位置阶段，Solver算整个身体的位置，以减少重叠和关节脱离。每个阶段都有自己的迭代次数，此外，如果误差较小，位置阶段可以提前退出迭代。建议的Box2D的迭代次数，**速度阶段为8，位置阶段为3**。调整迭代次数，是性能和精度之间的一个权衡，使用较少的迭代次数可以提高性能，但是也意味着损失精度。

> 时间步长和迭代次数完全无关

```C# 
float timeStep = 1 / 33;
int velocityIterations = 6;
int positionIterations = 2;
world.Step(timeStep, velocityIterations, positionIterations);
```

## 重要概念

### Contacts

`Contacts`是由Box2D创建的对象，用于管理两个载具之间的碰撞。如果载具有子代，比如链行，则每个相关的子代都有一个`Contacts`。即，碰撞时发生再两两之间的，也就是所谓的**AABB**概念。

> AABB树是由AABB包围盒节点构成的二叉树，常用于碰撞检测。树的每一个节点，都是一个包围盒，且节点的包围盒包裹了所有子节点的包围盒。AABB树是一颗满二叉树，对象只存在于叶节点种，父节点的载具包含了子节点的包围盒。

一次碰撞，Box2D会涉及以下概念：

* **接触点**：也称之为碰撞点，是两个形状开始重合的点。Box2D会尽量使用少的接触点，例如两个相同大小的圆，理论上的接触点只有一个。
* **接触法线**：是一个单位向量，从一个形状指向另一个形状，按照惯例法线是从载具A指向载具B。
* **分离度**: 分离度是穿透的反面，当行传重叠时，分离度就是负的。
* **接触流行**：两个凸形多边形之间的接触最多可能产生2个接触点。这两个点都使用相同的法线，所以它们被分组为一个接触流形，这是一个连续接触区域的近似值。
* **法线力**：是指在接触点上施加的、防止形状穿透的力。为方便起见，Box2D用脉冲工作。法线脉冲只是法线力乘以时间步长。
* **切线力**：是指在一个接触点产生，模拟摩擦的力。也被存储为一个脉冲。

Box2D试图重新使用一个时间步骤的接触力结果作为下一个时间步骤的初始猜测。Box2D使用接触ID来匹配不同时间步骤的接触点。这些ID包含几何特征指数，有助于区分一个接触点和另一个接触点。

当两个夹具的AABBs重叠时就会产生接触。有时碰撞过滤会阻止接触点的产生。当AABBs不再重叠时，接触就会被破坏。

#### Contacts事件

一个Contacts产生时，经过了四个事件：

1. `Begin Contact Event`：**当载具重叠时，就被调用**。传感器和非传感器都会被调用。这个事件只能在时间步长内发生。
2. `End Contact Event`: **当载具不再重叠时，被调用**。传感器和非传感器都会被调用。当一个物体被摧毁时，也可能调用此事件。所以，这个事件可能发生再时间步长之外。
3. `Pre-Solve Event`: **这个事件发生在碰撞检测之后，碰撞模拟之前**。每次通过碰撞处理，此事件就会被调用，可以在此事件种，选择禁止载具的碰撞。由于连续的碰撞检测，这个事件可能在一个时间步长内，多次调用。
4. `Post-Solve Event`: **模拟之后调用**，在此事件中可以获得碰撞之后的数据，同时提供了分离度。

> `Pre-Solve Event`事件中，很好确定点状的状态和速度。

> 不要在这些事件改变物理世界的游戏逻辑，因为可能会导致出现无法处理的无主指针对象。最好是通过缓存数据，在时间步长之后处理。

#### 碰撞过滤

当并不希望某些物体之间放生碰撞时，可以通过实现`IContactFilter`接口，来控制碰撞的发生。

```C#
   public interface IContactFilter
    {
        /// Return true if contact calculations should be performed between these two shapes.
        /// @warning for performance reasons this is only called when the AABBs begin to overlap.
        bool ShouldCollide(Fixture fixtureA, Fixture fixtureB);
    }
```

## 参考

* [官方文档](https://box2d.org/documentation/index.html)