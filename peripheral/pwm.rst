============
PWM控制器
============


    PWM控制器一共可以输出5路不同的PWM信号，每一路的高电平持续时间、低电平持续时间都可以分别独立进行配置。PWM控制器的时钟源频率也可以进行配置。

***************
【基本特性】
***************

PWM控制器的主要特性如下表所示：

- PWM控制器时钟源最高可达16MHz，时钟源频率可以进行配置。
- IO输出最高可达8MHz。（时钟源=16MHz , high_time=low_time=1）
- 每个PWM通道相互独立，可以分别配置高低电平持续时间，但是公用时钟源。
- 5路输出，每一路PWM对应的IO口可以进行配置。

***************
【时钟结构】
***************

PWM时钟由系统32MHz时钟生成，经过可配置的分频器之后，送往各路PWM输出。

分频参数CLK_DIV可以进行配置，CLK_DIV取值范围为0x00-0xFF，当CLK_DIV>0的时候，分频输出频率如下公式所示：

.. image:: img/pwm_clk.png

PWM 的时钟结构如图所示：

.. image:: img/pwm_freq.png



***************
【程序设计】
***************



第一步：设置分频参数
=============================

分频参数和PWM通道无关，改变分频参数会改变所有PWM通道。

所以不在每个通道的参数设置，而是在宏定义中进行设置。在“app_pwm.h”中，定义了分频参数以及数据结构。

.. code:: c

    #define ALL_CHANNEL_PWM_CLK_DIV     1


其中“ALL_CHANNEL_PWM_CLK_DIV”即CLK_DIV分频数值，取值范围是0x00-0xFF。参数取值与分频数值如下表所示：


==============================      =======================================
ALL_CHANNEL_PWM_CLK_DIV取值             分频
==============================      =======================================
0                                       1/2
0x01-0xFF                               1/(1+ ALL_CHANNEL_PWM_CLK_DIV)
==============================      =======================================


第二步：实例化PWM对象
==============================


.. code:: c

    app_pwm_inst_t bxpwm = PWM_INSTANCE();


如果使用PWM外设，必须首先声明PWM实例，然后才可以对实例进行操作。

其中“PWM_INSTANCE”宏定义可以进行PWM实例的声明。只需声明一个实例即可，一个实例中包含了5个channel。




第三步：设置PWM的IO口，并初始化
====================================


.. code:: c

    bxpwm.channel[PWM_CHANNEL_0].pin_num = 8;
    bxpwm.channel[PWM_CHANNEL_1].pin_num = 9;
    bxpwm.channel[PWM_CHANNEL_2].pin_num = 10;
    bxpwm.channel[PWM_CHANNEL_3].pin_num = 11;
    bxpwm.channel[PWM_CHANNEL_4].pin_num = 12;
    app_pwm_init(&bxpwm.inst);


如果用不到5个channel，则可以不进行配置对应的pin_num。

IO设置完毕之后，采用app_pwm_init函数来进行初始化。


第四步：开启或关闭输出：
==============================

方式一：配置高低电平
----------------------------

.. code:: c

    //#define ALL_CHANNEL_PWM_CLK_DIV     1
    app_pwm_set_time(&bxpwm.inst , PWM_CHANNEL_0 ,   1 ,   1);      //high:0.0625us     low:0.0625us     period:8MHz
    app_pwm_set_time(&bxpwm.inst , PWM_CHANNEL_1 , 160 , 160);      //high:10us         low:10us         period:50kHz
    app_pwm_set_time(&bxpwm.inst , PWM_CHANNEL_2 , 160 , 320);      //high:10us         low:20us         period:33.33kHz
    app_pwm_set_time(&bxpwm.inst , PWM_CHANNEL_3 , 160 ,   0);      //still high
    app_pwm_set_time(&bxpwm.inst , PWM_CHANNEL_4 ,   0 , 160);      //still low


如上述代码所示，参数含义如下表：

=============    ==================================================
参数               含义
=============    ==================================================
hdl                配置哪个PWM实例。
channel            配置哪一个通道
high_time          | 高电平持续时间，以分频之后的频率来计数。
                   | 如果该参数为0，则输出常低。
low_time           | 低电平持续时间，以分频之后的频率来计数。
                   | 如果该参数为0，则输出常高。
=============    ==================================================



方式二：配置频率与占空比
----------------------------

.. code:: c

    //#define ALL_CHANNEL_PWM_CLK_DIV     1
    app_pwm_set_duty(&bxpwm.inst , PWM_CHANNEL_0 , 1000   ,   0);   //still low
    app_pwm_set_duty(&bxpwm.inst , PWM_CHANNEL_1 , 1000   , 100);   //still high
    app_pwm_set_duty(&bxpwm.inst , PWM_CHANNEL_2 , 1000   ,  10);   //high:100us     low:900us    period:1000us
    app_pwm_set_duty(&bxpwm.inst , PWM_CHANNEL_3 , 10000  ,  20);   //high:20us      low:80us     period:100us
    app_pwm_set_duty(&bxpwm.inst , PWM_CHANNEL_4 , 100000 ,  30);   //high:3us       low:7us      period:10us


如上述代码所示，参数含义如下表：

=============    ==================================================
参数               含义
=============    ==================================================
hdl                配置哪个PWM实例。
channel            配置哪一个通道
frequency          输出PWM的频率，单位为Hz。(max 160k hz)
percent            | PWM的占空比，范围：0~100
                   | 如果该参数为100，则输出常高。
                   | 如果该参数为0，则输出常低。
=============    ==================================================












