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

## 创建并运行一个最基本的SUMO情景模拟

>基于SUMO的配置文件与输入文件都可通过**手动编写**来使用, 本章节将先使用SUMO的GUI组件**创建情景**, 并在保存后放出**相关文件**的代码内容.

首先在确保环境变量中有SUMO的可执行文件目录的情况下打开SUMO的`netedit`工具(通常情况下, Windows下的安装包与Ubuntu下的apt安装均已完成了环境变量的设置)

命令行中输入`netedit`并回车, 等待GUI界面打开.

![Open netedit](images/Image2022-03-22-02-23-44.png)

netedit的界面中因没有打开或创建的路网文件(network), 显示为空白.

### 创建路网结构(Road Network)

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

#### 创建节点(Node)与连接(Edge)

确保`创建并连接元素`工具已被选中, 并开始在下方界面中点击创建第一个节点, 并在随后在不同位置创建第二个节点. 在创建两个节点的同时, 两个节点(`node`)之间的连接(`edge`)会被自动创建

![Node Creation](images/Image2022-03-22-02-40-07.png)

如图所示, 路网中已经成功创建了两个节点, 并在两个节点中创建了一条道路作为连接. 在SUMO模拟中, 连接具有**方向性**(**directional**), 即道路拥有指定的起点与终点, 并具有单向性. 选中工具栏中的`检查元素属性`工具后选中图中已创建的道路或节点, 即可查看此对象的属性

![node Properties](images/Image2022-03-22-02-48-31.png)

![Edge Properties](images/Image2022-03-22-02-45-13.png)

在左侧的面板中可以得知这条道路(`edge`)的`id`为`E4`, 且起点`from`与终点`to`分别为`J10`与`J11`.

我们可以在左侧面板中设置节点(`node`)或道路(`edge`)的`id`以及其他属性, 包括设定节点的`pos`属性来调节该节点在路网中的**坐标位置**.

#### 创建对向道路(Reverse Edge)

如想在已创建道路的基础上添加一个对向的道路(`J11`到`J10`), 我们可以通过使用`创建并连接元素`工具依次点击`J11`节点与`J10`节点即可完成创建.

![Reverse Edge](images/Image2022-03-22-02-53-52.png)

如果想要图方便, 省去为每一条道路创建对向道路的步骤, 可在选中`创建并连接元素`工具的同时, 在工具栏右侧选中`自动创建对向道路`项(从右往左第一项).

![Auto Reverse](images/Image2022-03-22-02-56-32.png)

#### 连续创建节点与连接

如想连续创建多个连续的节点, 可在选中`创建并连接元素`工具后, 在工具栏右侧选中`一次性创建多个连续的节点`(从右往左第二项), 并开始在路网中创建节点.

![Successive Tool](images/Image2022-03-23-11-18-55.png)

连续的节点之间会自动创建连接, 并可通过按下`ESC`来中止连续创建.

![Successive Nodes](images/Image2022-03-23-11-18-21.png)

#### 保存路网结构

在完成对路网的创建后, 通过`File->Save Network As...`来将路网保存为一个`*.net.xml`格式的文件

下为通过`netedit`创建的一个具有3个节点, 2对相向道路的路网结构图及其保存后的文本文件内容.

![Sample Network](images/Image2022-03-23-11-29-11.png)

```xml
<?xml version="1.0" encoding="UTF-8"?>

<!-- generated on 2022-03-23 11:29:31 by Eclipse SUMO netedit Version 1.12.0
<configuration xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="http://sumo.dlr.de/xsd/netconvertConfiguration.xsd">

    <output>
        <output-file value="D:\Documents\Python\sample.net.xml"/>
    </output>

    <processing>
        <offset.disable-normalization value="true"/>
    </processing>

    <junctions>
        <no-turnarounds value="true"/>
    </junctions>

    <report>
        <aggregate-warnings value="5"/>
    </report>

</configuration>
-->

<net version="1.9" junctionCornerDetail="5" limitTurnSpeed="5.50" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="http://sumo.dlr.de/xsd/net_file.xsd">

    <location netOffset="0.00,0.00" convBoundary="-7.26,52.74,62.07,74.22" origBoundary="10000000000.00,10000000000.00,-10000000000.00,-10000000000.00" projParameter="!"/>

    <edge id=":J1_0" function="internal">
        <lane id=":J1_0_0" index="0" speed="4.41" length="1.04" shape="22.83,54.34 22.53,54.35 22.32,54.39 22.13,54.48 21.88,54.66"/>
    </edge>
    <edge id=":J1_1" function="internal">
        <lane id=":J1_1_0" index="0" speed="7.04" length="3.12" shape="19.97,52.09 20.72,51.56 21.30,51.28 21.93,51.16 22.85,51.14"/>
    </edge>

    <edge id="-E0" from="J1" to="J0" priority="-1">
        <lane id="-E0_0" index="0" speed="13.89" length="35.06" shape="21.88,54.66 -6.31,75.51"/>
    </edge>
    <edge id="-E1" from="J2" to="J1" priority="-1">
        <lane id="-E1_0" index="0" speed="13.89" length="39.24" shape="62.07,54.49 22.83,54.34"/>
    </edge>
    <edge id="E0" from="J0" to="J1" priority="-1">
        <lane id="E0_0" index="0" speed="13.89" length="35.06" shape="-8.21,72.94 19.97,52.09"/>
    </edge>
    <edge id="E1" from="J1" to="J2" priority="-1">
        <lane id="E1_0" index="0" speed="13.89" length="39.24" shape="22.85,51.14 62.08,51.29"/>
    </edge>

    <junction id="J0" type="dead_end" x="-7.26" y="74.22" incLanes="-E0_0" intLanes="" shape="-7.26,74.22 -5.36,76.79 -7.26,74.22"/>
    <junction id="J1" type="priority" x="21.78" y="52.74" incLanes="-E1_0 E0_0" intLanes=":J1_0_0 :J1_1_0" shape="22.83,55.94 22.85,49.54 21.32,49.60 20.79,49.72 20.29,49.94 19.73,50.29 19.02,50.80">
        <request index="0" response="00" foes="00" cont="0"/>
        <request index="1" response="00" foes="00" cont="0"/>
    </junction>
    <junction id="J2" type="dead_end" x="62.07" y="52.89" incLanes="E1_0" intLanes="" shape="62.07,52.89 62.09,49.69 62.07,52.89"/>

    <connection from="-E1" to="-E0" fromLane="0" toLane="0" via=":J1_0_0" dir="s" state="M"/>
    <connection from="E0" to="E1" fromLane="0" toLane="0" via=":J1_1_0" dir="s" state="M"/>

    <connection from=":J1_0" to="-E0" fromLane="0" toLane="0" dir="s" state="M"/>
    <connection from=":J1_1" to="E1" fromLane="0" toLane="0" dir="s" state="M"/>

</net>
```

其中`:J1_0`与`:J1_1`为因转弯幅度偏大, SUMO自动于中间节点J1的内部创建的低速车段, 其余的4个`edge`项目都为3个节点之间依次连接的双向车道, 以及`junction`项目即为路网中的3个节点.

> 注: 于SUMO中, `junction`与`node`同义, 即上文中`<junction></junction>`可在实际编写中用`<node></node>`代替

[官网参考 - SUMO_Road_Networks](https://sumo.dlr.de/docs/Networks/SUMO_Road_Networks.html)

#### 从OSM地图文件中获取路网文件

(TODO: 过于无脑, 先搁着)

### 创建路网流量

在完成了路网结构的创建后, 即可开始在路网中添加车辆实体并为其规划路线.

将顶部菜单栏中编辑模式切换为`Demand`.

![Demand Mode](images/Image2022-03-23-11-46-47.png)

在菜单栏下方工具栏中可以找到在为路网编辑流量时的常用工具

![Demand Tools](images/Image2022-03-23-14-08-31.png)

其中左侧4项的图标与功能都与路网编辑模式中相同，右侧2项分别为

* `创建路线`
* `创建交通工具`

> 此外还有包括但不限于 `创建行人`, `创建交通灯`等工具, 等有需要时再介绍

使用`创建路线`工具, 即可以在一个或多个指定节点连接(edge)上创建一个路线(route), 而后可以使用`创建交通工具`创建车辆实体并绑定在**已创建好的路线上**.

或者也可以直接使用`创建交通工具`工具的特定模式, 直接在创建交通实体(vehicle)的同时指定路线(route), SUMO将自动创建路线实体并与交通实体绑定.

如果同一路线将被多个交通实体绑定, 那推荐使用`创建路线`工具先创建好路线, 再与交通实体绑定. 如果单一路线的使用率不高, 可能只有少数车辆实体通行, 那推荐使用`创建交通工具`工具快速创建车辆实体与对应路线实体.

因为这个章节为基础示例, 下属子章节中将直接使用`创建交通工具`来创建车辆.

#### 创建车辆实体的不同工具

选中`创建交通工具`, 左侧面板即会显示相对应的设置选项

![Vehicle Options](images/Image2022-03-23-14-31-44.png)

其中, `Vehicles`项默认将设为`trip`

![Vehicles Option List](images/Image2022-03-23-14-32-52.png)

可选的选项代表了不同的交通工具创建方式:

* 创建路程(trip)实体(指定起点与终点edge)
* 创建路程(trip)实体(指定起点与重点junction)
* 创建交通工具(vehicle)实体(指定已有route)
* 创建交通工具(vehicle)实体(内嵌route)
* 创建交通流(flow)(指定起点与终点edge)
* 创建交通流(flow)(指定起点与终点junction)
* ...

`trip`与`flow`在SUMO文档中被统称为`待补全路线`(Incomplete Route), 即没有完整的路线定义, 只需指定起点与终点, 中间的途径路线将由SUMO自行判断并补全.

相对的, `vehicle`项创建出的交通工具需要绑定一个完整的路线(Route).

在实际应用中, `trip`适用于快速创建一个交通工具实体, `flow`适用于创建一个指定起点终点的**车辆流**, 即在特定时间内有一定数量相同类型的交通工具生成并完成路线, 省去了重复使用`vehicle`或`trip`创建车辆实体的工作.

此处先用`trip`进行简单的车辆创建与演示, 在章节末将显示`trip`, `flow`与`vehicle`在生成代码上的区别, 以便根据实际使用情况自行编写`*.rou.xml`文件.

#### 使用`trip`创建车辆实体

在上文中提到的`Vehicles`项中选择默认的`trip(from-to edges)`, 并选择将要创建的车辆类型`Parent vType`.

![Vehicle Type](images/Image2022-03-23-16-15-50.png)

选择不同的车辆类型将继承不同的预定义车辆属性(最大速度, 加速度等). 之后的内容里会提及如何创建自定义车辆类型. 在这里先选择预设的车辆类型`DEGFAULT_VEHTYPE`.

![Veh Prop List](images/Image2022-03-23-16-18-34.png)

在底下的面板里可以调整创建车辆的其他属性, 这里维持默认.

在右侧路网中依次点击路线的起终点, 后点击左侧面板的`完成路线创建(Finish route creation)`

![Finish Route Creation](images/Image2022-03-23-16-39-52.png)

完成后即可看到车辆在路网中被绘制

![Trip Sample](images/Image2022-03-23-16-40-59.png)

#### 保存车流

点击`File->Demand Elements->Save Demand Elements As...`来保存车流为`*.rou.xml`后缀的文件

以下为创建的车流路线及其生成的代码内容

![Trip Sample](images/Image2022-03-23-16-43-37.png)

```xml
<?xml version="1.0" encoding="UTF-8"?>

<!-- generated on 2022-03-23 16:43:28 by Eclipse SUMO netedit Version 1.12.0
-->

<routes xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="http://sumo.dlr.de/xsd/routes_file.xsd">
    <trip id="t_0" depart="0.00" from="E0" to="E1"/>
</routes>
```

#### `trip`, `vehicle`, `flow`间的代码内容对比

以下为分别使用创建`创建路程(trip)实体(指定起点与终点edge)`, `创建交通工具(vehicle)实体(内嵌route)`, `创建交通流(flow)(指定起点与终点edge)`, 创建流程都与上文类似, 与特定模式相关的**特殊参数**(如`flow`中的车流量相关的参数设置)将于后面间的章节解释.

```xml
    <flow id="f_0" begin="0.00" color="green" from="E0" to="E1" end="3600.00" vehsPerHour="1800.00"/>

    <trip id="t_0" depart="0.00" from="E0" to="E1"/>

    <vehicle id="v_0" depart="0.00" color="red">
        <route edges="E0 E1"/>
    </vehicle>
```

[参考官网 - Definition of Vehicles, Vehicle Types and Routes](https://sumo.dlr.de/docs/Definition_of_Vehicles%2C_Vehicle_Types%2C_and_Routes.html)

### 在SUMO-GUI中运行模拟

在`netedit`中确保**路网文件被读取**以及**车流文件被读取**的情况下点击`Edit->Open in sumo-gui` (同时`Edit`下的`Load additionals in sumo-gui`与`Load demand in sumo-gui`需要被选中, 否则车流数据**无法传递**至SUMO-GUI)

![SUMO-GUI](images/Image2022-03-23-17-17-42.png)

在开始操作前先点击`File->Save Configuration`, 将模拟配置保存为`*.sumocfg`文件, 后续便无需通过`netedit`打开`SUMO-GUI`, 只需在`SUMO-GUI`中读取此文件.

以下为保存的`*.sumocfg`的文件内容

```xml
<?xml version="1.0" encoding="UTF-8"?>

<!-- generated on 2022-03-23 17:28:42 by Eclipse SUMO GUI Version 1.12.0
-->

<configuration xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="http://sumo.dlr.de/xsd/sumoConfiguration.xsd">

    <input>
        <net-file value="D:\Documents\Python\sample.net.xml"/>
        <route-files value="D:\Documents\Python\sample.rou.xml"/>
    </input>

</configuration>
```

像之前解释的一样, `*.sumocfg`文件中存储了路网文件`net-file`与车流文件`route-files`的文件位置.

回到SUMO-GUI, 在顶部工具栏中调整模拟的步进延迟(Delay)为合适数值(即经过多长时间模拟自动进行下一步), 此处设为50ms.

![Delay](images/Image2022-03-23-17-21-09.png)

点击左侧的`开始模拟(Start the loaded simulation)`按钮, 即可开始演示并等待模拟完成.
(或点击`模拟步进(Perform a single simulation step)`按钮完成单次步进).

![Simulation](images/Image2022-03-23-17-25-00.png)

车辆会在模拟开始时显示在路网中

>注: SUMO-GUI的车辆显示**默认为小三角**, 如需更换为`netedit`中一样的**车辆图形**, 可以打开`Edit->Edit Visualization`中的`Vehicles`标签页更改

待模拟完成后, 提示窗口弹出.

![Sim End](images/Image2022-03-23-17-27-56.png)

## 使用Python的Traci包与SUMO进行基本的通信与控制

Traci API 为SUMO提供的一套可用于调用与操控SUMO情景模拟的接口, 于包括但不限于Python, C++的各种编程语言中均可使用.

根据功能的划分, Traci API被划分为了13个子模组, 分别为:

* gui
* lane
* poi
* simulation
* trafficlight
* vehicletype
* edge
* inductionloop
* junction
* multientryexit
* polygon
* route
* person
* vehicle

由于本章节只涉及对先前创建的路网模拟情景进行通信与操控, 下文里只有`simulation`与`vehicle`子模组将会被调用. 其余模组以及更细致的使用场景将于后面的章节中介绍.

### 搭建可使用Traci的Python环境

由于Python的环境搭建与代码验证较为快捷, 此部分内容将使用Windows下的`Python 3.9.7`进行演示.

SUMO在安装时默认会配置好可被Python调用的traci包, 一般情况下可直接在Python终端内导入.

![Traci](images/Image2022-03-25-14-52-34.png)

在Python终端内使用`import traci`命令尝试导入traci包. 如果导入失败, 则尝试手动使用`pip`安装traci:

```sh
pip install traci
```

### 通过Traci启动SUMO-GUI

在Python中导入了traci后, 便可使用`traci.start()`命令来启动SUMO.
在系统环境中已经配置了SUMO启动文件目录的前提下(即命令行中输入`sumo.exe`或`sumo-gui.exe`可顺利启动SUMO程序), 在Python终端中输入

```python
traci.start(['sumo-gui','-c','sample.sumocfg'])
```

其中`sample.sumocfg`为上文中保存的`sumocfg`配置文件的**相对路径**或**绝对路径**.

若配置正常, SUMO的GUI界面将被启动, 同时`sample.sumocfg`将被正常读取.

![traci.start](images/Image2022-03-25-15-03-40.png)

如果`sumocfg`文件与之前章节中展示的代码内容一样或相似, 会发现**Python终端进入等待返回的状态, 无法进行任何命令输入**, 且**SUMO无法正常进行模拟**(即模拟与步进无法正常完成).

为修复这一问题, 需要先关闭Python终端且退出SUMO, 打开`sumocfg`文件并在`configuration`项目下添加`gui_only`项:

```xml
<?xml version="1.0" encoding="UTF-8"?>

<!-- generated on 2022-03-23 17:28:42 by Eclipse SUMO GUI Version 1.12.0
-->

<configuration xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="http://sumo.dlr.de/xsd/sumoConfiguration.xsd">

    <input>
        <net-file value="D:\Documents\Python\sample.net.xml"/>
        <route-files value="D:\Documents\Python\sample.rou.xml"/>
    </input>

    <gui_only>
        <start value="true"/>
    </gui_only>

</configuration>
```

保存后再根据上文中的步骤重新调用`traci.start()`命令, Python终端会正常打印返回值并进入等候命令状态.

![traci.start - gui](images/Image2022-03-25-15-14-57.png)

输入`help(traci.start)`查看其命令文档, 可查看更详细的使用说明

```python
>>> help(traci.start)
Help on function start in module traci.main:

start(cmd, port=None, numRetries=60, label='default', verbose=False, traceFile=None, traceGetters=True, stdout=None, doSwitch=True)
    Start a sumo server using cmd, establish a connection to it and
    store it under the given label. This method is not thread-safe.

    - cmd (list): uses the Popen syntax. i.e. ['sumo', '-c', 'run.sumocfg']. The remote
      port option will be added automatically
    - numRetries (int): retries on failing to connect to sumo (more retries are needed
      if a big .net.xml file must be loaded)
    - label (string) : distinguish multiple traci connections used in the same script
    - verbose (bool): print complete cmd
    - traceFile (string): write all traci commands to FILE for debugging
    - traceGetters (bool): whether to include get-commands in traceFile
    - stdout (iostream): where to pipe sumo process stdout
```

其中显示traci与SUMO的连接会被赋予一个为`default`的`label`参数, 如果想要同时启动多个SUMO情景模拟, 可通过手动为`traci.start`中的`label`参数赋值, 并在后续命令中指明`label`连接来单独操控:

```python
traci.start(['sumo-gui','-c','another_sample.sumocfg'], label='another_instance')
```

### 通过Traci正式开始模拟并获取信息

此时GUI界面中`Delay`栏旁的计时器仍处于空值, 即模拟仍未正式启动, 在Python终端中输入`traci.simulationStep()`, 即可命令SUMO启动模拟, 我们先前设置好的车流也将出现在路网中.

![Simstart](images/Image2022-03-25-15-29-02.png)

>`traci.simulationStep()`函数与上文中介绍GUI内容时的**步进按钮**效果相同, 且会返回一个内容为空的`list`: `[]`. `traci.simulationStep()`的返回值实际上为traci在SUMO模拟中对特定信息的**订阅**(`subscription`), 即用户希望traci在模拟中想要了解的信息的值, 如车辆的速度, 车辆的数量等. 与`subscription`相关的内容将在基本操作介绍完后提及.

`traci.simulationStep()`**默认执行一次步进**, 如果想一次性执行多次步进, 可在命令中附加参数, SUMO模拟将连续执行步进直至模拟时间与参数值相同, 如`traci.simulationStep(5)`即一次性执行多次步进, **直至模拟时间达到5**.

同时, 如果想在Python中得知当前模拟时间信息, 可使用`traci.simulation.getTime()`函数.

```python
>>> traci.simulation.getTime()
1.0
>>> traci.simulationStep(5)
[]
>>> traci.simulation.getTime()
5.0
>>>
```

此时, 可通过traci中vehicle子模组下的函数来获取模拟中车辆的各类相关信息.
如`traci.vehicle.getIDList()`可获取目前存在于路网中所有交通工具实体的id,
`traci.vehicle.getSpeed()`可使用先前获取的id信息, 查询特定车辆的速度信息,
`traci.vehicle.getLaneID()`可得知特定车辆目前处于的道路等.

```python
>>> traci.vehicle.getIDList()
('f_0.0', 'f_0.1')
>>> traci.vehicle.getSpeed('f_0.0')
8.21320034653414
>>> traci.vehicle.getLaneID('f_0.0')
'E0_0'
```

在使用完毕后, 使用`traci.close()`停止SUMO的模拟并关闭与SUMO间的连接.

### 对traci接口的处理逻辑做轻量定制

SUMO中的Traci接口的工作逻辑是通过调用一组Python脚本实现的. 我们同时也可以通过修改Traci的脚本来实现对Traci接口的工作逻辑做些许更改与定制.

找到SUMO的安装目录, 并参照先前提到的SUMO安装目录结构找到`Sumo/tools/traci`文件夹. 文件夹下列举了Traci API所用到的一系列脚本文件.

以`vehicle`子模组为例, 打开上述文件夹下的`_vehicle.py`文件, 在`VehicleDomain`的class下可以看见traci中可调用的各类函数.
在`getSpeed()`函数下添加一行`print("Hello there!")`的代码.

![_vehicle.py](images/Image2022-03-25-16-59-55.png)

保存文件后再一次用traci打开SUMO, 并在模拟开始后使用`traci.vehicle.getSpeed()`函数, 可以看出刚才对脚本做出的修改已经生效.

```python
>>> traci.vehicle.getSpeed('f_0.0')
Hello there!
8.21320034653414
```

## 使用SUMO组件自动创建路网及车流文件

除了使用`netedit`或手动编辑文本的方式创建路网文件以及车流文件外, SUMO还提供了一系列命令行工具以及脚本可以帮助我们创建路网文件以及车流文件

### 使用`netgenerate`工具快速生成路网

`netgenerate`为SUMO提供的根据用户需求参数快速创建所需简易路网文件的命令行工具.
通常`netgenerage`在安装SUMO时已经被添加进了PATH路径内, 在终端内输入`netgenerage`或`netgenerate.exe`(Windows)便可直接调用.

`netgenerage`目前支持创建三种类型的路网结构, 分别为:

* **网格路网**(Grid Network)
* **蛛网路网**(Spider Network)
* **随机路网**(Random Network)

在使用`netgenerate`构建路网时与路网构建相关的参数有:

| 参数                      | 定义                                                                  | 默认值    |
| ------------------------- | --------------------------------------------------------------------- | --------- |
| --grid, -g                | 指定路网结构为网格路网                                                | false     |
| --grid.number             | 指定网格路网中横向与纵向的全局节点数量                                | 5         |
| --grid.length             | 指定网格路网中横向与纵向的全局连接长度                                | 100       |
| --grid.x-number           | 指定网格路网中横向节点的数量(可覆盖全局数量)                          | 5         |
| --grid.y-number           | 指定网格路网中纵向节点的数量(可覆盖全局数量)                          | 5         |
| --grid.x-length           | 指定网格路网中横向连接的长度(可覆盖全局长度)                          | 5         |
| --grid.y-length           | 指定网格路网中纵向连接的长度(可覆盖全局长度)                          | 5         |
| --grid.attach-length      | 指定网格路网中横向与纵向的边缘连接的全局长度(长度为0代表边缘没有连接) | 0         |
| --gird.x-attach-length    | 指定网格路网中横向的边缘连接长度(可覆盖全局长度)                      | 0         |
| --gird.y-attach-length    | 指定网格路网中纵向的边缘连接长度(可覆盖全局长度)                      | 0         |
| --spider, -s              | 指定路网结构为蛛网结构                                                | false     |
| --spider.arm-number       | 指定蛛网结构中轴的数量                                                | 13        |
| --spider.circle-number    | 指定蛛网结构的层级数量                                                | 20        |
| --spider.space-radius     | 指定蛛网结构的层间距离                                                | 100       |
| --spider.omit-center      | 是否忽略蛛网结构的中心节点                                            | false     |
| --rand, -r                | 指定路网结构为随机路网                                                | false     |
| --rand.iterations         | 指定随机路网中往路网添加连接的次数                                    | 2000      |
| --bidi-probability        | 指定路网中添加双向车道的几率                                          | 1         |
| --rand.max-distance       | 指定随机路网中单条连接的最大长度                                      | 250       |
| --rand.min-distance       | 指定随机路网中单条连接的最小长度                                      | 100       |
| --rand.min-angle          | 指定随机路网中每组车道间的最小夹角度数                                | 45        |
| --rand.connectivity       | 随机路网中新建车道不为死路的概率                                      | 0..95     |
| --rand.neighbor-dist<1-6> | 随机路网中节点拥有正好<1-6>条连接的概率                               | Undefined |
| --rand.grid               | 将随机路网以网格的方式构建, 并以<rand.min-distance>参数为网格间的间距 | false     |

以下为使用以上部分参数创建的不同类型的路网文件

* `netgenerate.exe --grid --grid.number=8 --grid.length=20 --grid.attach-length=50 -o grid-eight.net.xml`
![gird-eight](images/Image20220411202631.png)
* `netgenerate.exe --spider --spider.arm-number=5 --spider.circle-number=7 --spider.space-radius=20 -o spider-five.net.xml`
![spider-five](images/Image20220411202943.png)  
* `netgenerate.exe --rand --rand.iterations=100 --rand.max-distance=50 --rand.min-distance=10 --rand.min-angle=30 -o rand-hundred.net.xml`
![rand-hundred](images/Image20220411203202.png)

[官网参考 - NetGenerate](https://sumo.dlr.de/docs/netgenerate.html)

### 使用`randomTrips.py`自动生成车流

`randomTrips.py`为SUMO官方提供的方便用户创建大量车流的工具性Python脚本, 位于SUMO安装目录下的`tools`文件夹中.

在使用`randomTrips.py`脚本时, 有以下常用参数可以被指定

| 参数               | 定义                                                             | 默认值    |
| ------------------ | ---------------------------------------------------------------- | --------- |
| -n                 | 用于生成车流文件的路网文件(network)                              | Undefined |
| -b                 | 车流开始生成的时间(秒)                                           | 0         |
| -e                 | 车流停止生成的时间(秒)                                           | 3600      |
| -p                 | 车流生成的频率(秒/辆)                                            | 1         |
| --random-depart    | 在开始时间与结束时间之间随机发车                                 | Undefined |
| --binomial \<INT\> | 使车辆在开始与结束时间之间的生成频率符合参数为\<INT\>的二项分布  | Undefined |
| --prefix           | 生成车流的id前缀                                                 | ""        |
| --min-distance     | 车流从出发点至到达点之间的最短距离                               | Undefined |
| --flows \<INT\>    | 将生成的车流整合为\<INT\>个`flow`, 使之符合`-p`定义的发车频率    | Undefined |
| --random           | 设置此值为`true`使相同参数可以生成随机的车流, 否则为相同车流     | false     |
| --seed             | 车流生成的种子值                                                 | Undefined |
| --route-file       | 使trip文件通过后台调用`duarouter`生成指定文件名的车流文件(route) | Undefined |

值得注意的是, `randomTrips.py`默认并不会生成完整的车流文件, 而是定义只包含**起点**与**终点**的`trips`文件, 即之前讲到的**待补全路线(Incomplete Route)**. `randomTrips.py`所生成的`trips`并**不保证**起点与终点之间可以被连通.
如果想要一步到位生成完整的车流文件(route), 需要使用到上表中提到的`--route-file`参数, 该参数将使脚本在后台调用SUMO的车流生成工具`duarouter`生成从起点到终点之间的**完整路线**.

以下命令将以上表中的部分参数生成一份示例`trips`文件, 因内容过多故不在此粘贴.
`python <Path to SUMO>\tools\randomTrips.py' -n <network>.net.xml -b 0 -e 360 -p 1.5 --prefix="tp-" -o trips.trips.xml`

更多的参数可以参照:

* `python  <Path to SUMO>\tools\randomTrips.py' --help`
* [官网参考 - randomTrips.py](https://sumo.dlr.de/docs/Tools/Trip.html)

### 使用`jtrrouter`自动生成完整的车流

在上面提到, `randomTrips.py`并不会生成完整的车流文件, 除非使用`--route-file`参数来调用SUMO的车流生成工具`duarouter`以生成完整车流.

`duarouter`是基于 **最短路径算法(Shortest Path Computation)** 生成完整的车流路线, 同时SUMO还提供了另一种生成完整车流路线的工具, 即`jtrrouter`, 其基于 **各个节点往不同方向转向的概率(Junction Turning Ratio)** 来生成完整车流路线

> 具体使用方法待补全, 感兴趣可参考[官方文档 - jtrrouter](https://sumo.dlr.de/docs/jtrrouter.html)

## 在路网模拟中添加行人(Pedestrain)

在添加行人之前, 先建造好一个带有三车道及逆向车道的十字路口路网并在十字中心添加人行横道

![crossroad](images/Image20220418212350.png)
> 需要注意的是, 十字路口的中心也需要设置为一个节点, 否则横竖两条重叠的路线将不被视为交叉

### 在十字路口中添加人行横道

在`Network`编辑模式下, 选择工具栏中的`人行道编辑工具`
![Cross-editing tool](images/Image20220418212819.png)

在添加人行横道之前, 需要先点击交叉路口的**中心节点**, 使四个方向的道路全部变为绿色, 以便程序知道人行横道需要被添加到哪个路口.
![Select Center](images/Image20220418213103.png)

随后依次点击左侧两个方向的车道作为人行横道需要穿过的道路, 再点击左侧面板中的`Create Crossing`按钮以创建人行横道.
![First Cross](images/Image20220418213535.png)  

再重复以上步骤, 为交叉路口创建完整的人行横道.
![Complete-Corss](images/Image20220418213648.png)  

在`net`文件中, 人行横道也被视为一种`连接(edge)`, 不过与普通连接不同的是, 人行横道执行的是被成为`crossing`的函数. 因为上述操作保存的`net`文件过长, 以下仅展示与人行横道有关的内容

```xml
<edge id=":J2_c0" function="crossing" crossingEdges="E2 -E2">
        <lane id=":J2_c0_0" index="0" allow="pedestrian" speed="1.00" length="6.40" width="4.00" shape="3.20,5.20 -3.20,5.20"/>
    </edge>
    <edge id=":J2_c1" function="crossing" crossingEdges="E1 -E1">
        <lane id=":J2_c1_0" index="0" allow="pedestrian" speed="1.00" length="6.40" width="4.00" shape="5.20,-3.20 5.20,3.20"/>
    </edge>
    <edge id=":J2_c2" function="crossing" crossingEdges="E3 -E3">
        <lane id=":J2_c2_0" index="0" allow="pedestrian" speed="1.00" length="6.40" width="4.00" shape="-3.20,-5.20 3.20,-5.20"/>
    </edge>
    <edge id=":J2_c3" function="crossing" crossingEdges="-E0 E0">
        <lane id=":J2_c3_0" index="0" allow="pedestrian" speed="1.00" length="6.40" width="4.00" shape="-5.20,3.20 -5.20,-3.20"/>
    </edge>
    <edge id=":J2_w0" function="walkingarea">
        <lane id=":J2_w0_0" index="0" allow="pedestrian" speed="1.00" length="10.51" width="3.20" shape="-3.20,7.20 0.00,7.20 0.00,7.20 3.20,7.20 7.20,3.20 7.20,0.00 7.20,0.00 7.20,-3.20 3.20,-7.20 0.00,-7.20 0.00,-7.20 -3.20,-7.20 -7.20,-3.20 -7.20,0.00 -7.20,0.00 -7.20,3.20 -3.31,5.98 -3.64,4.98 -4.20,4.20 -4.98,3.64 -5.98,3.31"/>
    </edge>
    <edge id=":J2_w1" function="walkingarea">
        <lane id=":J2_w1_0" index="0" allow="pedestrian" speed="1.00" length="2.83" width="4.00" shape="-3.20,3.20 -3.20,7.20 -7.20,3.20 -3.20,3.20"/>
    </edge>
    <edge id=":J2_w2" function="walkingarea">
        <lane id=":J2_w2_0" index="0" allow="pedestrian" speed="1.00" length="2.83" width="4.00" shape="3.20,3.20 7.20,3.20 3.20,7.20 3.20,3.20"/>
    </edge>
    <edge id=":J2_w3" function="walkingarea">
        <lane id=":J2_w3_0" index="0" allow="pedestrian" speed="1.00" length="2.83" width="4.00" shape="3.20,-3.20 3.20,-7.20 7.20,-3.20 3.20,-3.20"/>
    </edge>
    <edge id=":J2_w4" function="walkingarea">
        <lane id=":J2_w4_0" index="0" allow="pedestrian" speed="1.00" length="2.83" width="4.00" shape="-3.20,-3.20 -7.20,-3.20 -3.20,-7.20 -3.20,-3.20"/>
    </edge>
    <edge id=":J3_w0" function="walkingarea">
        <lane id=":J3_w0_0" index="0" allow="pedestrian" speed="1.00" length="3.20" width="3.20" shape="100.00,0.00 100.00,3.20 100.00,-3.20 100.00,0.00"/>
    </edge>
    <edge id=":J4_w0" function="walkingarea">
        <lane id=":J4_w0_0" index="0" allow="pedestrian" speed="1.00" length="3.20" width="3.20" shape="0.00,100.00 -3.20,100.00 3.20,100.00 0.00,100.00"/>
    </edge>
    <edge id=":J5_w0" function="walkingarea">
        <lane id=":J5_w0_0" index="0" allow="pedestrian" speed="1.00" length="3.20" width="3.20" shape="0.00,-100.00 3.20,-100.00 -3.20,-100.00 0.00,-100.00"/>
    </edge>
```

### 添加行人

在保存好十字路口路网后切换至`Demand`编辑模式, 在工具栏中选择`创建行人工具`
![PersonCreatingTool](images/Image20220419002214.png)  

随后在路网中选择两条`连接`作为行人的出发点与到达点, 随后点击左侧面板的`Finish Route Creation`按钮完成添加.
![Person Creation](images/Image20220419002413.png)  

创建好一个或多个行人路线后, 通过上文中保存车流文件的相同方法保存,或是通过`Ctrl+Shift+D`快捷保存至`*.rou.xml`文件.

在`*.rou.xml`文件中, 行人被以`person`项目记录, 并附有此行人的路线项目`personTrip`.

```xml
<routes xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="http://sumo.dlr.de/xsd/routes_file.xsd">
    <person id="p_1" depart="0.00">
        <personTrip from="E0" to="-E2"/>
    </person>
</routes>
```

>注: 经个人测试发现, 若在netedit中删除行人及路线, 会在`*.rou.xml`中残留类似于`<person id="p_1" depart="0.00">`的空项目, 此项目会导致SUMO在读取`*.rou.xml`文件时报错, 需手动删除.

### 在SUMO中运行行人的模拟

与上文中的车流章节类似, 在完成对行人流的保存后可通过`Edit->Open in SUMO-gui`打开SUMO模拟并自动加载行人流. 在设置好`delay`后点击运行即可看到行人在路网中的行走过程.

>注: 在路网规模过大的情况下, 通常行人的显示模型会因为太小而完全找不到行人, 可通过更改工具栏中`Edit->Edit Visualization`项目下的`Persons`菜单中`Exaggerated by ...`数值放大行人模型的显示, 或在模拟开始后使用`Locate->Locate Persons`工具手动寻找行人位置.

![Person Walking](images/Image20220419003850.png)  

通过模拟我们可以看出, SUMO中的行人模拟会自行通过设定好的起点与终点规划路线, 不受车道方向约束(毕竟人行道也没有得逆行), 会在道路**最靠边**的"行人专有道路"上行走, 并且只会通过我们设定好的**人行横道**穿越马路.

## 在路网模拟中添加交通信号灯(Traffic Light)

## 设定车辆的跟驰模型(Car-Following Model)

## 创建新的交通工具实体类型

## 使用Veins与Traci接口进行交互
