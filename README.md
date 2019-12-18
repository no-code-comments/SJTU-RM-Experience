# Introduction

本人萌新一枚，此次随着同级大佬参加SJTU校内Robomaster比赛，并一人负责电控部分。毕竟我仅仅对于写代码比较熟练，对于电机设备，型号，接线这类知识一窍不通，导致遇到了很多困难，也踩了无数的坑，甚至连累了一些机械方面的设计（菜是原罪）。这份文件记录一下我的收获，也可能可以为一些像我一样的零基础小白一些建议。

（如果有缘能够看见这篇文档，也可能可以避开一些不必要的麻烦。）

# Declaration

本篇文档中的代码直接使用**SJTU交龙战队**提供的原框架，以及**人参果**大佬在此基础上作出的改进版本，在此向其表示感谢。

未修改版本的框架文件存储在**RM_Frame**文件夹中。

# Contents

[TOC]

# Body

## 框架简述

其实作为SJTU校内赛来说，电控部分并不需要阅读源码框架，因为并不需要对其进行大规模的修改，重构，也不需要对其底层逻辑、基本原理理解得很透彻。总的来说，电控的主要工作量还是在学习新知识上，并不是在设计算法上。

校内赛的框架算是搭建的比较完整，层次良好。文档注释虽然欠缺，但主要核心部分还是比较完备的。如果觉得需要哪一块内容却又读不懂，又没有足够的文档或注释去解释，那么基本一定是**你方向错了，这一块不用改**。

在进入下一节**使用框架**之前，我们得先了解一下RM主控板以及驱动程序是如何搭建、烧录并工作的。一个大致的流程是先使用**STM32**搭建框架，再使用**Keil**进行详细的代码撰写，最后使用**STLink**将程序烧录到主控板中，最后主控板连接电源和电路元件进行驱动。

### STM32

全名STM32CubeMX，大疆开发板A型所对应的主控板代号是STM32F427IIHx。STM32可以被想象成一个代码生成器，他的作用是提前生成一段代码供给Keil继续编程操作，所以STM32的唯一操作就是**配置**+**生成**，然后就没它的事了。

我们主要需要在STM32上配置主控板中需要的引脚输出。每一个元件插在主控板上的位置都需要在STM32中进行配置，只不过很多引脚框架已经配置好了而已。

另一个重点的配置就是时钟。一般不会需要太过复杂的时钟配置，校内赛已经配置好了最基础的时钟，而我又不太会，就不赘述了。

### Keil

这款软件是一款IDE，用来撰写程序，编译并使用STLink烧录进主控板。这款软件唯一无可替代的就是烧录功能，其它功能可以用其它IDE代替，因为RM主控板的驱动程序是由C语言书写的，Keil本质上也是个C语言IDE。

### 主控板

RM的主控板的各个引脚在说明书中有详细的解释，值得一提的是，在说明书末尾部分，有各个引脚在STM32中所对应的代号，这个十分重要。

- 电源输入
  - STLink烧录端口：3.3V电源输入，可以提供大部分LED的电压。
  - USB连接端口：5V电源输入，可以提供遥控器接收端的电压，可以调试遥控器是否能够连接。
  - 24V电源输入端口：24V电源输入，比赛及调试时一般（一定）使用此端口供电。
- 可控电源输出端口：一共有四个，实际操作中直接使用，可以提供24V电压。
- CAN数据线端口：一般只使用2-pin CAN数据线端口，用于3508和2006的数据交互，后面会讲。
- PWM端口：产生PWM波。一共有20个PWM端口，校内赛框架一共配置了同一周期(20ms)四个端口**PA0**, **PA1**, **PA2**, **PA3**。如果需要其它的端口需要自行手动在STM32中配置。
- DBUS端口：连接遥控器接收器。
- STLink烧录端口：连接电脑烧录程序。

## 使用框架

### 第一步：准备

首先我们先使用校内赛的原框架来驱动一下机械车，好让自己有一点成就感。在此之前，我们需要一个搭建好底盘和麦轮，固定了M3508电机的小车。显然这不是我们电控的事情。

主控板需要提前烧录好我们的校内赛框架（没错，啥都不用动，直接烧进去）。常见的bug是无法检测到STLink驱动器，这时可能是线连错了（STLink的四条线按照主控板说明和STLink上的标识一一对应），或者可能是STLink驱动没有装（那就去装驱动）。

除此之外，我们需要准备好我们的材料。

- 4个3508电机（在车上）。
- 4个C620电调。
- 4条7-pin数据线，4条2-pin CAN数据线
- 电调中心板
- 10-pin转电源&2pin数据线（一头是10-pin，一头是电源线和2-pin CAN数据线）
- 烧录好源程序的主控板
- 智能电池（去借）和电池架

### 第二步：连线

首先连接电机M3508和电调C620。先将它们自带的三相电源线连在一起，还需要7-pin数据线进行连接，四个电机电调均需如法炮制。

再将C620电调的自带的电源线连接在电调中心板上，把电调上标有“CAN”接口的橡胶剥掉，使用2-pin CAN数据线连接C620和电调中心板。

使用10-pin电源线连接电调中心板的10-pin接口，与主控板上的24V电源接口和旁边的CAN接口。

智能电池插入电池架，并将电调中心板上的另一个电源接口连接电池架上的对应接口。

最后需要在主控板DBUS接口上插上接收器。

### 第三部：初始化

开启智能电池的方式为：短按一下，再长按2秒，并切换电池架上的开关按钮。主控板通电后会点亮绿色和红色LED灯泡，没亮说明有问题。

我们需要先对遥控器与接收器进行配对。再依次设置C620的ID号。具体操作方法有相关教程，没有就Google。

C620的ID号会影响麦轮的行为方式，所以不正确的ID会导致不正确的行为模式。具体对应ID我并没有研究过，重要的是让它动起来，如果发现有问题了，那么多试几次一定会成功的。

设置完电调ID后需要重启电源，遥控器左上右上都拨至最高的那个档位（没错就是那个**OFF**），然后极其柔和地操作摇杆，如果小车有反应，那么就成功了。如果小车反应比较大，请把**右上档位拨至最下而不是直接关掉遥控器**！

### 第四部：你成功了

去吃点的东西犒劳一下自己，也可以打几局游戏。

## 修改框架

一般来说，所有需要修改的程序仅仅只在文件`FunctionTask.c`中，以下讲解如无特殊声明，均在`FunctionTask.c`中完成。

### 紧急制停

我们首先需要在原框架中加上几行。

在`ControlTask.c`中82行，添加

```c
for(int i=0;i<8;i++) {InitMotor(can1[i]);InitMotor(can2[i]);}
// 添加部分
setCAN11();
setCAN12();
setCAN21();
setCAN22();
//------------
```

在`CanMotor.c`中337行，添加

```c
(id->Handle)(id);
// 添加部分
id->Intensity = 0;
//------------
```

### 遥控模式

先说右上角的三个档位，最上为遥控器模式，中间为键鼠模式，最下为紧急制停，所以一般来说只会用到最高的档位。

左上角的三个档位分别为三个状态，源码中的整体结构为

```c
if(WorkState == NORMAL_STATE)
{
    // ...
}
if(WorkState == ADDITIONAL_STATE_ONE)
{
    // ...
}
if(WorkState == ADDITIONAL_STATE_TWO)
{
    // ...
}
```

最上代表NORMAL_STATE，中间代表ADDITIONAL_STATE_ONE，左下代表ADDITIONAL_STATE_TWO。

所以遥控器上的标识和实际效果没有任何关系-_-||

### 底盘移动——M3508电机

底盘的移动仅仅依赖于四个3508的不同驱动方式。源代码为

```c
ChassisSpeedRef.forward_back_ref = channelrcol * RC_CHASSIS_SPEED_REF;
ChassisSpeedRef.left_right_ref   = -channelrrow * RC_CHASSIS_SPEED_REF/2;
rotate_speed = -channellrow * RC_ROTATE_SPEED_REF;
```

可以看出

- `ChassisSpeedRef.forward_back_ref`控制小车前后移动
- `ChassisSpeedRef.left_right_ref`控制小车左右平动
- `rotate_speed`控制小车左右旋转

#### 修改

理论上来说，这点代码足够操作了，因为它涵盖了所有移动操作。如果需要调整速度，那么可以乘上一个常数；如果需要调整左右手，那么修改`channellcol`, `channellrow`, `channelrcol`, `channelrrow`这四个变量即可。

其中一点是我的队员需要在控制小车时，如果同时左旋并且倒退，需要达到向左后方向的效果。（正常来说，如果倒退并且左转，是向右后方向）。如果要实现这种操作，只需要在倒退时将旋转方向反向。

```c
if(ChassisSpeedRef.forward_back_ref > 0) rotate_speed = -rotate_speed;
```

另外一点是如果队员喜欢飙车，可以强行设定一个加速度限制（\^v\^)

```c
// 声明全局变量并不是一个好方法，更推荐静态变量
// 但是爽啊
int speed_ref
int original_speed = 0;
const int ACCELERATION = 7;
```

- `original_speed`用来存储上一次获得的速度。
- `ACCELERATION`用来限制最大加速度
- `speed_ref`用以保存过滤后的数据

然后就可以把源码改成这样，以前进后退为例

```c
// 检测是否超过加速度阈值
if(channellcol > 0 && channellcol - original_speed >= ACCELERATION) speed_ref = original_speed + ACCELERATION;
else if(channellcol < 0 && channellcol - original_speed <= -ACCELERATION) speed_ref = original_speed - ACCELERATION;
else speed_ref = channellcol;

// 使用speed_ref而不是channelrcol控制前进后退速度
ChassisSpeedRef.forward_back_ref = speed_ref * RC_CHASSIS_SPEED_REF;
```

觉得加速度太大或太小，只需要调整`ACCELERATION`的值就可以了。

### M2006电机

2006电机连线方式与3508一模一样，需要搭配C610电调，ID号也不能与C620的ID重复。源码为

```c
M2006.TargetAngle += channellcol * 0.2;
```

使用校内赛框架会发现M2006转速极慢，不管如何改变乘子，都无法提速，所以需要修改代码。

#### 修改

在文件`CanMotor.c`中，修改对于M2006电机的初始配置，将对应部分修改为如下代码

```c
MotorINFO M2006 = Normal_MOTORINFO_Init(36.0,&ControlNM,
                  fw_PID_INIT(10, 0.0, 0.0,   15000.0, 15000.0, 15000.0, 15000.0),
                  fw_PID_INIT(30, 0.0, 0.0,   15000.0, 15000.0, 15000.0, 15000.0));
```

```c
void ControlNM(MotorINFO* id)
{
 if(id==0) return;
 if(id->s_count == 0)
 {  
  uint16_t  ThisAngle; 
  double   ThisSpeed; 
  ThisAngle = id->RxMsgC6x0.angle;    //未处理角度
  if(id->FirstEnter==1) {id->lastRead = ThisAngle;id->FirstEnter = 0;return;}
  if(ThisAngle<=id->lastRead)
  {
   if((id->lastRead-ThisAngle)>3000)//编码器上溢
    id->RealAngle = id->RealAngle + (ThisAngle+8192-id->lastRead) * 360 / 8192.0 / id->ReductionRate;
   else//正常
    id->RealAngle = id->RealAngle - (id->lastRead - ThisAngle) * 360 / 8192.0 / id->ReductionRate;
  }
  else
  {
   if((ThisAngle-id->lastRead)>3000)//编码器下溢
    id->RealAngle = id->RealAngle - (id->lastRead+8192-ThisAngle) *360 / 8192.0 / id->ReductionRate;
   else//正常
    id->RealAngle = id->RealAngle + (ThisAngle - id->lastRead) * 360 / 8192.0 / id->ReductionRate;
  }
  ThisSpeed = id->RxMsgC6x0.RotateSpeed * 6 / id->ReductionRate;  // °/s
  
  id->Intensity = PID_PROCESS_Double(&(id->positionPID),&(id->speedPID),id->TargetAngle,id->RealAngle,ThisSpeed);
  id->s_count = 0;
  id->lastRead = ThisAngle;
  //id->Intensity =5000;
 }
 else
 {
  id->s_count++;
 }  
}
```

经过此番修改，M2006电机可以以较快的速度进行转动。

另外，在修改完代码后，如果给予2006电机一个很快的速度（实测大约对应`M2006.TargetAngle += 30;`），M2006会不受控制地进行旋转，并且速度更快，但是无法通过常规操作停止或控制(`M2006.TargetAngle += value;`不起作用)。此时可以通过另一种方法强行停止，代码为`M2006.TargetAngle = M206.RealAngle;`。暂未明白其原理。

因此，如果对于M2006的转速有一定要求，又不需要可变转速的情况下，可以采用如下代码

```c
// 通过motor_2006_flag控制，初始化为1，即开机直接处于正转状态
int motor_2006_flag = 1;
// 通过motor_2006_control_flag锁死状态，避免一次推杆过程发生多次不可控状态切换。
int motor_2006_control_flag = 1;

// ...
if(WorkState <= 0) return;
// 控制代码放在判断工作状态之前，使其不论在哪个工作状态都能够旋转。
// M2006电机控制
if(1 == motor_2006_flag) M2006.TargetAngle += 60;
else if(-1 == motor_2006_flag) M2006.TargetAngle -= 60;
else M2006.TargetAngle = M2006.RealAngle;

// ...
// ADDITIONAL_STATE_ONE中
// 右摇杆向上，若毛刷不转，则控制其正转；否则使其停止转动。
if(channelrcol >= 300 && 1 == motor_2006_control_flag) {
    if(0 == motor_2006_flag) motor_2006_flag = 1;
    else if(1 == motor_2006_flag || -1 == motor_2006_flag) motor_2006_flag = 0;
    motor_2006_control_flag = 0;
}
// 右摇杆向下，若毛刷不转，则控制其反转；否则使其停止转动。
else if(channelrcol <= -300 && 1 == motor_2006_control_flag) {
    if(0 == motor_2006_flag) motor_2006_flag = -1;
    else if(-1 == motor_2006_flag || 1 == motor_2006_flag) motor_2006_flag = 0;
    motor_2006_control_flag = 0;
}
// 等到右摇杆重新回到中心区域，刷新状态
else if(abs(channelrcol) <= 150) {
    motor_2006_control_flag = 1;
}
```

然后，然后就没了。

### 舵机

一般来说，舵机是以PWM信号进行驱动的，并且PWM信号周期为20ms。校内赛框架中PWM信号(TIM2)配置为19ms，影响不大。

一般的舵机接收0.5ms-2.5ms高电平脉冲，对应了

- 0.5ms：0度（反转）
- 1.5ms：90度（不转）
- 2.5ms：180度（正转）

提供代码为

```c
HAL_TIM_PWM_Start(&htim2,TIM_CHANNEL_2);
__HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_2, 1500);
```

第一个参数为计时器，我们需要用到校内赛框架配置好的第二个计时器，所以参数为`&htim2`，对应的通道为第二个，所以参数为`TIM_CHANNEL_2`，而函数`__HAL_TIM_SET_COMPARE()`的第三个参数为高电平的时间，单位为微秒，所以源代码对应生成了1.5ms脉冲。

#### 修改

一个简单的控制程序如下

```c
// 初始状态为1.5ms，对应不转
int pulse_time = 1500
HAL_TIM_PWM_Start(&htim2, TIM_CHANNEL_2);
// 左摇杆往上打正转
if(channellcol >= 300) pulse_time = 2500;
// 左摇杆往下打反转
if(channellcol <= -300) pulse_time = 500;
__HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_2, pulse_time);
```

如果想要让舵机一直施力，也可以参考以上M2006的控制代码。

需要注意的是，由于，所以舵机的PWM信号线不能像M2006一样随便哪一个口都行，而是要在STM32中看好对应的端口代码，在主控板说明书上找到对应的引脚，最后把PWM数据线插在对应的引脚上，才能够正常驱动。如果需要更多的舵机，那么一般只需要`TIN_CHANNEL_3`, `TIM_CHANNEL_4`。这三个通道已经配置好，并且如果需要超过三个，需要手动配置STM32。

#### 简单时钟配置

打开STM32，在`Pinout&Configuration`中左边栏，点击`Timers`

我们需要选择一个可以生成PWM波的始终，以TIM4为例，点击`TIM4`

`Mode`部分：`Clock Source`设置为`Internal Clock`，`Channel1` 设置为 `PWM Generation CH1`

`Configuration`部分：点击`Parameter Settings`，`Prescaler`设置为`84-1`，`Counter Period`设定为想要的周期减一，比如说要5ms周期就设定为`5000-1`

STM32会自动设置一个引脚，在右侧`Pinout view`中查看或修改。下方搜索栏搜索`TIM4`可以发现默认引脚为`PD12`

如果确认使用该引脚，那么右键点击它，选择`Signal Pinning`

最后点击上方的`GENERATE CODE`，若不成功，可能是安装有问题。

最后可以在Keil中编写代码了。此时这个输出端口就是参数`&htim4`和`TIM_CHANNEL_1`了。因为重新进行了配置，再次编译需要将所有文件重新编译，编译时间会和第一次编译一样，需要2分钟左右。

# Tips

对于我踩过的一些坑，作了一些总结：

- 先尝试驱动麦轮（Body->使用框架），会增加很多对框架的熟悉程度。
- 电机与电调的搭配如果不是大疆的比较复杂，并且电调有双向单向之分。部分电调只能驱动电机往一个方向转。
- 没有必要去弄明白框架的架构，浪费时间还毫无卵用。
- 谨慎使用步进电机，可能是我水平受限，步进电机实际操作比较复杂，并且可能会出力矩不够导致不断抖动的问题。可能的解决方式是使用可变频率的脉冲，但是配置过程我没有深究。
- 麦克纳姆轮自上而下看应该是个X型，否则移动可能正确，但是阻力，振动都非常大，操作很受限制，对于小车磨损也会很严重。
- 任何操作的人都应该要知道紧急制停的模式。

# Conclusion

虽然最后结果不咋地，做出来的事也不多，但是还是混了一点经验，也稍微了解了一下电控的职能。不多去尝试尝试，怎么能知道自己原来有那么菜呢？

最后非常感谢陪我一起参加比赛的朋友们。
