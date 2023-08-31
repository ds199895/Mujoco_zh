---
layout:
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
---

# Unity Plug-in

## Introduction 简介

MuJoCo Unity插件允许Unity编辑器和运行时使用MuJoCo物理引擎。用户可以导入MJCF文件并在编辑器中编辑模型。该插件在大多数方面依赖Unity - 资源、游戏逻辑、模拟时间 - 但使用MuJoCo来确定物体如何移动，使开发者可以访问MuJoCo的完整API。

## **Installation instructions安装**

该插件目录（可在https://github.com/deepmind/mujoco/tree/main/unity 上找到）包括一个package.json文件。Unity的包管理器会识别此文件，并将插件的C#代码库导入到您的项目中。此外，Unity还需要MuJoCo的本地库，可以在https://github.com/deepmind/mujoco/releases 的特定平台存档中找到。

在Unity版本2020.2及更高版本中，包管理器在导入包时会查找本地库文件并将其复制到包目录中。或者，您可以手动将本地库复制到包目录并重新命名，参见下面的平台特定说明。该库还可以复制到项目的Assets目录下的任何位置。

### Mac OS

在使用本地库之前，必须至少运行一次MuJoCo应用程序，以将库注册为可信二进制文件。然后，从/Applications/MuJoCo.app/Contents/Frameworks/mujoco.framework/Versions/Current/libmujoco.2.3.8.dylib（可以通过浏览MuJoCo.app的内容找到它）复制动态库文件，并将其重命名为mujoco.dylib。

### Linux

将tar.gz存档文件展开到\~/.mujoco目录。然后，从\~/.mujoco/mujoco-2.3.8/lib/libmujoco.so.2.3.8复制动态库，[并将其重命名为libmujoco.so](http://xn--libmujoco-fp6nt8zl5huobt41dolikq8s.so/)。

### Windows

将zip存档文件展开到名为MuJoCo的目录中，该目录位于您的用户目录下，然后复制文件MuJoCo\bin\mujoco.dll

## **Using the plug-in 使用插件**

### Importer导入

导入器是从编辑器的Asset菜单中调用的：点击“Import MuJoCo Scene”，然后选择包含模型的MJCF规范的XML文件。

### **Context menus上下文菜单**

右键单击geom组件提供了两个选项：

1. “Add mesh renderer（添加网格渲染器）”会向渲染geom的同一游戏对象添加组件：一个标准的MeshRenderer和一个MjMeshFilter，它会在geom形状属性更改时重新创建一个程序化的网格。
2. “Convert to free object（转换为自由对象）”会添加两个新的游戏对象：一个带有MjBody组件的父对象和一个带有MjFreeJoint组件的兄弟对象。这允许先前的静态geom在场景中自由移动。**此操作仅适用于“world” geom - 即当前没有MjBody父对象的geom**。

右键单击Unity Collider 提供了一个选项：“添加匹配的MuJoCo geom”到同一游戏对象。请注意，这并不包括物理的完整转换 - 刚体(Rigidbody)、关节配置(ArticulationBody)仍需要手动重新创建。

### **Mouse spring鼠标拖动**

当所选的游戏对象具有MjBody组件时，可以通过在Scene视图中执行控制-左拖动操作，将弹簧力施加到该物体朝向鼠标光标。弹簧力的起始点的三维位置是通过将鼠标位置投影到由摄像机X方向和世界Y方向定义的平面上来确定的。按下Shift键将改变投影平面，使其与世界的X和Z轴平行。

### **Tips to Unity users建议**

如果遇到任何编译或运行时错误，系统的状态将是不确定的。因此，我们建议在控制台窗口中打开“错误暂停”。

在PhysX中，每个Rigidbody都是一个“自由体”。相反，**MuJoCo需要明确指定关节以实现可移动性**。为了方便起见，我们提供了一个上下文菜单，用于通过添加父MjBody和兄弟MjFreeJoint来“释放”world geom（即没有任何MjBody父组件的MjGeom组件）。

该插件不支持没有物理存在的碰撞检测，因此没有内置的触发器碰撞器的概念。可以通过添加触摸传感器并读取其SensorReading值来读取接触力的存在或不存在（这将对应于法线力，详见触摸传感器文档\<sensor-touch>）

### Design principles设计原则

插件设计提供了MJCF元素与Unity组件之间的一对一映射。为了使用MuJoCo模拟Unity场景（例如，当用户在编辑器中点击“Play”按钮时），插件的工作流包括：

1. 扫描场景中的游戏对象层次结构，寻找MuJoCo组件。
2. **创建一个MJCF描述**并传递给MuJoCo的编译器。
3. 通过MuJoCo**数据结构中的相应索引**将每个组件绑定到MuJoCo运行时。这个索引用于在模拟过程中更新Unity的变换。

这一设计原则有一些重要影响：

* 大多数Unity组件的字段直接对应于MJCF属性。因此，用户可以参考MuJoCo文档以获取有关不同值语义的详细信息。
* MuJoCo组件在GameObject层次结构中的布局决定了生成的MuJoCo模型的布局。因此，我们采用了一个设计规则，即**每个游戏对象最多只能有一个MuJoCo组件**。
* 我们依赖于Unity来进行空间配置，这需要对矢量组件进行重新排列，因为Unity使用左手坐标系，以Y作为垂直轴，而MuJoCo使用右手坐标系，以Z作为垂直轴。
* Unity的变换缩放会影响整个游戏对象子树的位置、方向和比例。然而，MuJoCo不支持倾斜的圆柱体和胶囊体（通过椭球体原型支持倾斜的球体）的碰撞。对于geoms和sites的操作工具会忽略这种倾斜（类似于PhysX碰撞体），并始终显示原始形状，就如它将在物理中出现一样。
* 在运行时，更改组件字段的值**不会触发场景重新创建**，因此不会立即影响物理模拟。但是，新的值将在下一次场景重新创建时加载。

在尽可能的情况下，我们采用Unity的方式进行操作：**重力从Unity的物理设置中读取**，模拟步骤从Unity的时间管理器的固定时间步长中读取。外观的所有方面（例如，网格、材质和纹理）都由Unity的资产管理器处理，并使用材质资产进行RGBA规格的设置。

### Implementation notes注意事项

#### Importer workflow导入工作流

当用户选择一个MJCF文件时，导入器首先将文件加载到MuJoCo，将其保存到临时位置，然后处理生成的保存文件。这会产生一些影响：

1. 它验证了MJCF文件 - 我们确保保存的MJCF与模式匹配。
2. 它验证了资产（材质、网格、纹理）并将这些资产导入Unity，同时为geom的RGBA规格创建新的材质资产。
3. 它允许导入器处理\<include>元素，而无需复制MuJoCo的文件系统工作流程。
4. 即使原始模型使用geoms隐式定义了**物体的惯性，**当前版本的MuJoCo依然会生成带有明确的\<inertial>元素的MJCF文件。如果您计划更改导入模型的geom属性，请手动删除这些自动生成的MjInertial组件。我们计划在未来的MuJoCo版本中解决这个问题。

在Unity中，没有等同于MJCF的“级联”子句。因此，在Unity中，组件反映了应用所有相关默认类后的相应元素状态，并且原始MJCF中的类结构被丢弃。

#### The MuJoCo Scene Mujoco Scene组件

当创建一个MuJoCo场景时，MjScene组件首先扫描场景中的所有MjComponent实例。每个组件都使用Unity场景的空间结构创建自己的MJCF元素，以描述模型的初始参考姿势（在MuJoCo中称为qpos0）。MjScene根据各自游戏对象的层次结构合并这些XML元素，并创建一个单一的物理模型的MJCF描述。然后，它**创建了运行时结构mjModel和mjData**，并通过识别其唯一索引将每个组件绑定到运行时。

在运行时，MjScene.FixedUpdate() 调用 **mj\_step**，然后根据绑定时的索引 **MjComponent.MujocoId** 同步每个游戏对象的状态。只有在场景中包含任何MuJoCo组件时，应用程序启动时（例如，用户点击“播放”按钮时）才会自动添加MjScene组件。如果您的应用程序初始化阶段涉及在添加游戏对象和组件时进行物理模拟，您可以在初始化阶段结束后调用MjScene.CreateScene()。

场景**重新创建**通过以下方式维护物理和状态的连续性：

1. 关节的位置和速度被缓存。
2. MuJoCo的状态被重置（到qpos0），并且同步Unity的变换。
3. 生成一个新的XML，创建一个模型，其关节的qpos0与之前保持一致。
4. MuJoCo状态（对于保持不变的关节）从缓存中设置，并且同步Unity的变换。

因为MuJoCo库目前尚未暴露出**用于场景编辑的API**，所以添加和移除MuJoCo组件会导致完整的场景重新创建。对于大型模型或频繁发生这种情况可能会很昂贵。我们预计在将来的MuJoCo版本中将解决这个性能限制。

#### Global Settings全局设置

一个例外是全局设置组件（Global Settings component）。该组件负责所有包含在MJCF的固定大小、**单例**全局元素中的配置选项。目前，它包含与\<option>和\<size>元素对应的信息，**未来如果\<compiler>元素中的字段与Unity插件相关，则也将用于\<compiler>元素。**

#### Invoking the importer at application runtime应用运行时唤醒导入器

导入器是由类MjImporterWithAssets实现的，它是MjcfImporter的子类。这个父类接受一个MJCF字符串并生成组件的层次结构。它可以在播放时调用（不涉及编辑器功能），并且不调用MuJoCo库的任何函数。这在MuJoCo模型以程序方式生成（例如，通过某种进化过程）和/或仅导入以进行转换（例如，到PhysX或URDF）时非常有用。**由于它不能与Unity的AssetManager交互**（这是编辑器的一个功能），这个类的功能受到限制。具体来说：

1. 它忽略所有资产（包括碰撞网格）。
2. 它忽略可视化部分（包括RGBA规格）。

#### MuJoCo sensor components MuJoCo传感器组件

MuJoCo定义了许多传感器，我们担心为每个传感器创建一个单独的MjComponent类会导致大量的代码重复。因此，我们根据被测量属性的对象类型（actuator/body/geom/joint/site）以及被测量数据的类型（scalar/vector/quaternion），创建了不同的类.

这是一个将类型映射到传感器的表格：

| **Mojoco对象类型** | **数据类型**   | **传感器名**                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| -------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Actuator       | Scalar     | <ul><li><code>actuatorpos</code></li><li><code>actuatorvel</code></li><li><code>actuatorfrc</code></li></ul>                                                                                                                                                                                                                                                                                                                                                       |
| Body           | Vector     | <ul><li><code>subtreecom</code></li><li><code>subtreelinvel</code></li><li><code>subtreeangmom</code></li><li><code>framepos</code></li><li><code>framexaxis</code></li><li><code>frameyaxis</code></li><li><code>framezaxis</code></li><li><code>framelinvel</code></li><li><code>frameangvel</code></li><li><code>framelinacc</code></li><li><code>frameangacc</code></li></ul>                                                                                  |
| Body           | Quaternion | <ul><li><code>framequat</code></li></ul>                                                                                                                                                                                                                                                                                                                                                                                                                           |
| Geom           | Vector     | <ul><li><code>framepos</code></li><li><code>framexaxis</code></li><li><code>frameyaxis</code></li><li><code>framezaxis</code></li><li><code>framelinvel</code></li><li><code>frameangvel</code></li><li><code>framelinacc</code></li><li><code>frameangacc</code></li></ul>                                                                                                                                                                                        |
| Geom           | Quaternion | <ul><li><code>framequat</code></li></ul>                                                                                                                                                                                                                                                                                                                                                                                                                           |
| Joint          | Scalar     | <ul><li><code>jointpos</code></li><li><code>jointvel</code></li><li><code>jointlimitpos</code></li><li><code>jointlimitvel</code></li><li><code>jointlimitfrc</code></li></ul>                                                                                                                                                                                                                                                                                     |
| Site           | Scalar     | <ul><li><code>touch</code></li><li><code>rangefinder</code></li></ul>                                                                                                                                                                                                                                                                                                                                                                                              |
| Site           | Vector     | <ul><li><code>accelerometer</code></li><li><code>velocimeter</code></li><li><code>force</code></li><li><code>torque</code></li><li><code>gyro</code></li><li><code>magnetometer</code></li><li><code>framepos</code></li><li><code>framexaxis</code></li><li><code>frameyaxis</code></li><li><code>framezaxis</code></li><li><code>framelinvel</code></li><li><code>frameangvel</code></li><li><code>framelinacc</code></li><li><code>frameangacc</code></li></ul> |
| Site           | Quaternion | <ul><li><code>framequat</code></li></ul>                                                                                                                                                                                                                                                                                                                                                                                                                           |

以下是相同的表格，反向映射传感器到UnityC#类：

| **传感器名**        | **插件类**                              |
| --------------- | ------------------------------------ |
| `accelerometer` | SiteVector                           |
| `actuatorfrc`   | ActuatorScalar                       |
| `actuatorpos`   | ActuatorScalar                       |
| `actuatorvel`   | ActuatorScalar                       |
| `force`         | SiteVector                           |
| `frameangacc`   | \*Vector (depends on frame type)     |
| `frameangvel`   | \*Vector (depends on frame type)     |
| `framelinacc`   | \*Vector (depends on frame type)     |
| `framelinvel`   | \*Vector (depends on frame type)     |
| `framepos`      | \*Vector (depends on frame type)     |
| `framequat`     | \*Quaternion (depends on frame type) |
| `framexaxis`    | \*Vector (depends on frame type)     |
| `frameyaxis`    | \*Vector (depends on frame type)     |
| `framezaxis`    | \*Vector (depends on frame type)     |
| `gyro`          | SiteVector                           |
| `jointlimitfrc` | JointScalar                          |
| `jointlimitpos` | JointScalar                          |
| `jointlimitvel` | JointScalar                          |
| `jointpos`      | JointScalar                          |
| `jointvel`      | JointScalar                          |
| `magnetometer`  | SiteVector                           |
| `subtreeangmom` | BodyVector                           |
| `subtreecom`    | BodyVector                           |
| `subtreelinvel` | BodyVector                           |
| `torque`        | SiteVector                           |
| `touch`         | SiteScalar                           |
| `velocimeter`   | SiteVector                           |

以下传感器尚未实现：

* tendonpos
* tendonvel
* ballquat
* ballangvel
* tendonlimitpos
* tendonlimitvel
* tendonlimitfrc
* user

#### Mesh Shapes

该插件允许在MuJoCo碰撞中使用任意的Unity网格。在模型编译时，MuJoCo调用qhull来创建网格的凸包，并将其用于碰撞检测。**目前，计算的凸包在Unity中不可见**，但我们计划在未来的版本中将其公开。

#### Interaction with External Processes

Roboti的MuJoCo插件为Unity中的模拟步骤使用了外部Python进程，并仅使用Unity进行渲染。相比之下，我们的插件依赖于Unity来进行模拟步骤。可以在外部进程“驱动”模拟的情况下使用我们的插件，例如通过设置qpos、调用mj\_kinematics、同步变换，然后使用Unity进行渲染或计算游戏逻辑。为了与外部进程建立通信，您可以使用Unity的ML-Agents包。
