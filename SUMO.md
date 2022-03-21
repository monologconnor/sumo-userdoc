# SUMO

> SUMO (Simulation of Urban MObility), 一个由Eclipse维护的开源道路仿真模拟应用, 提供了多种交通运输及包括路人行为场景模拟的解决方案, 此文档为个人在学习SUMO使用期间个人对SUMO原理及应用的理解的总结.

## SUMO 安装注意事项

经个人测试, SUMO在Windows的Linux子系统, WSL2(Windows Subsystem of Linux)下, SUMO的图形交互应用`sumo-gui`会出现**路网结构无法正常显示**的问题, 而同时因个人有使用OmNet++下Veins框架通过traci接口与SUMO进行接口通信的需求, Windows下OmNet++的Veins依赖包(Simu5G)**无法正常编译**, 只能通过WSL使用OmNet++, 且Windows下的SUMO与WSL2下的Veins间因WSL2的Hyper-V虚拟机性质, **无法正常通信**. 

为避免绕弯路, **推荐使用**Ubuntu系统或预装有OmNet++与Veins框架的Ubuntu虚拟机

## SUMO 文件结构(File Structure)

此部分将对*SUMO应用自身的文件结构*以及*SUMO工程文件目录下文件*进行分析

### SUMO应用自身文件结构

此部分以SUMO在Windows环境下的目录结构(`../Eclipse/Sumo/`)为例, SUMO安装目录下的文件夹有

* `./bin` - SUMO自身及附带工具的可执行文件
* `./data` - SUMO的图形界面所需要的图像材质数据等
* `./doc` - 包含SUMO官方的使用范例(examples), Python包文档(pydoc), 基础教学用范例(tutorial)及html用户文档(userdoc)
* `./include` - 包含libsumo的C Header文件
* `./share` - (未知)
* `./tools` - SUMO应用运行时所调用的Python脚本文件, 包括traci接口

如果要更改traci接口在与其他应用通信时的行为表现, 即需更改`./tools/traci`目录下的脚本文件

### SUMO工程目录下文件

SUMO在模拟时需要读取的文件种类偏多, 其中以下三种文件满足了SUMO最基本的情景模拟需求

* `*.sumocfg` - 模拟配置文件(configuration)
  里面记录了此SUMO交通模拟所需的输入文件(input), 运行时间管理(time), 以及GUI需求设置(gui_only)
* `*.net.xml` - 路网结构数据(road network)
  其中记录了SUMO模拟中的地图数据, 基本由节点(node)以及连接(edge)组成, 可通过SUMO附带的**netconvert**工具, 由osm地图数据转化得到, 也可由用户自己通过编辑此文件或通过SUMO的**netedit**组件, 在GUI界面中设计并保存得到
* `*.rou.xml` - 车流及路由数据(route)
  定义了场景模拟中会涉及的交通工具类型(vType), 路程内容(route), 交通工具实体(vehicle)以及在路程上行驶的车辆行程实体(trip)

在通过SUMO模拟道路场景时, SUMO通过读取`sumocfg`文件了解到此次模拟的相关配置, 并获取相关输入文件的位置(如路网文件`*.net.xml`及路由文件`*.rou.xml`)

以下为一个SUMO配置文件内容的一个示例.

```xml
<?xml version="1.0" encoding="UTF-8"?>

<!-- generated on 2022-03-10 16:29:46 by Eclipse SUMO GUI Version 1.12.0
-->

<configuration xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="http://sumo.dlr.de/xsd/sumoConfiguration.xsd">

    <input>
        <net-file value="D:\Documents\Python\simple.net.xml"/>
        <route-files value="D:\Documents\Python\simple.rou.xml"/>
    </input>

    <time>
        <begin value="0"/>
        <end value="1000"/>
        <step-length value="0.1"/>
    </time>

    <gui_only>
        <start value="true"/>
    </gui_only>

</configuration>
```

>注: 如需要在Python中通过traci接口启动SUMO的GUI界面, 需如示例所示, 将配置文件中`gui_only`下`start`项的值设置为`true`, 否则在使用`traci.start()`启动SUMO的GUI界面时, 即使GUI界面加载成功, Python终端仍无法返回`traci.start()`的值, 且无法进行任何进一步交互.

除以上三种基本的文件类型外, SUMO仍拥有多种输入文件类型可以被模拟配置文件`*.sumocfg`中的`input`项目所引用, 作为输入文件的补充(additional input), 如

* `*.flow.xml` - 与`*.rou.xml`相似, 都用于定义车辆实体
  而`*.rou.xml`只能对车辆实体进行逐一定义, 而`*.flow.xml`可以通过指定起点,终点与车辆数量, 一次性生成大量车辆实体, 适合当交通情景模拟需要创建**大量车辆**时使用
* `*.trip.xml` - 类似于`*.rou.xml`
  定义了交通工具实体的行程, 不过与`*.rou.xml`不同的是, `trip`项目只需指定行程的起点与终点, 中间途径的节点`node`会又SUMO自动补全, 而`*.rou.xml`必须指定交通工具实体于行程中经过的所有节点
* `*.turns.xml` - 定义SUMO中路口选择的概率

> 据个人经验, SUMO输入文件中定义内容与文件类型的限制并不严苛, 如`*.trip.xml`中定义的内容也可在`*.rou.xml`中被定义, 如在`*.rou.xml`文件中可定义一个trip 项目为: `<trip id="t_0" depart="0.00" ...>`. 同理`flow`项目也可在`*.rou.xml`中被定义

更多文件类型可以参考[官网](<https://sumo.dlr.de/docs/Other/File_Extensions.html>)

### 创建一个最基本的SUMO情景模拟

>基于SUMO的配置文件与输入文件都可通过**手动编写**来使用, 本章节将先使用SUMO的GUI组件**创建情景**, 并在保存后放出**相关文件**的代码内容.

首先在确保环境变量中有SUMO的可执行文件目录的情况下打开SUMO的`netedit`工具(通常情况下, Windows下的安装包与Ubuntu下的apt安装均已完成了环境变量的设置)

命令行中输入`netedit`并回车, 等待GUI界面打开.

![Open netedit](images/Image2022-03-22-02-23-44.png)

netedit的界面中因没有打开或创建的路网文件(network), 显示为空白.

在File项中选择创建新网络`New Network`

![New Network](images/Image2022-03-22-02-25-56.png)

路网编辑的第一步即在路网中创建第一个地点(节点).
确保菜单栏中的编辑模式已设置为`Network`

![Network edit mode](images/Image2022-03-22-02-28-57.png)

在`Network`模式下的工具栏中, 本章节中或需用到的常用工具从左到右依次为:

* `检查元素属性`
* `删除元素`
* `元素复选`
* `移动元素`
* `创建并连接元素`

![Edit tools](images/Image2022-03-22-02-36-58.png)

确保`创建并连接元素`工具已被选中, 并开始在下方界面中点击创建第一个节点, 并在随后在不同位置创建第二个节点. 在创建两个节点的同时, 两个节点(`node`)之间的连接(`edge`)会被自动创建

![Node Creation](images/Image2022-03-22-02-40-07.png)

如图所示, 路网中已经成功创建了两个节点, 并在两个节点中创建了一条道路作为连接. 在SUMO模拟中, 连接具有**方向性**(**directional**), 即道路拥有指定的起点与终点, 并具有单向性. 选中工具栏中的`检查元素属性`工具后选中图中已创建的道路或节点, 即可查看此对象的属性

![node Properties](images/Image2022-03-22-02-48-31.png)

![Edge Properties](images/Image2022-03-22-02-45-13.png)

在左侧的面板中可以得知这条道路(`edge`)的`id`为`E4`, 且起点`from`与终点`to`分别为`J10`与`J11`.

我们可以在左侧面板中设置节点(`node`)或道路(`edge`)的`id`以及其他属性, 包括设定节点的`pos`属性来调节该节点在路网中的**坐标位置**.

如想在已创建道路的基础上添加一个对向的道路(`J11`到`J10`), 我们可以通过使用`创建并连接元素`工具依次点击`J11`节点与`J10`节点即可完成创建.

![Reverse Edge](images/Image2022-03-22-02-53-52.png)

如果想要图方便, 省去为每一条道路创建对向道路的步骤, 可在选中`创建并连接元素`工具的同时, 在工具栏右侧选中`自动创建对向道路`项(从右往左第一项).

![Auto Reverse](images/Image2022-03-22-02-56-32.png)

