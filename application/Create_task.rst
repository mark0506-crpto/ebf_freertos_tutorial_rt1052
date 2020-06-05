.. vim: syntax=rst

创建任务
============

在上一章，我们已经基于野火I.MX RT系列开发板创建好了FreeRTOS的工程模板，这章开始我们将真正进入如何使用FreeRTOS的征程，先从最简单的创建任务开始，点亮一个LED，以慰藉下尔等初学者弱小的心灵。

硬件初始化
~~~~~~~~~~

本章创建的任务需要用到开发板上的LED，所以先要将LED相关的函数初始化好，为了方便以后统一管理板级外设的初始
化，我们在main.c文件中创建一个BSP_Init()函数，专门用于存放板级外设初始化函数，具体见 代码清单:14_1_ 的高亮部分。

.. code-block:: c
    :caption: 代码清单:14_1 BSP_Init()中添加硬件初始化函数
    :emphasize-lines: 31, 36, 40
    :name: 代码清单:14_1
    :linenos:

    /***********************************************************************
    * @ 函数名： BSP_Init
    * @ 功能说明：
    板级外设初始化，所有板子上的初始化均可放在这个函数里面
    * @ 参数：
    * @ 返回值：无
    *********************************************************************/
    static void BSP_Init(void)
    {
    /* 初始化内存保护单元 */
        BOARD_ConfigMPU();
    /* 初始化开发板引脚 */
        BOARD_InitPins();
    /* 初始化开发板时钟 */
        BOARD_BootClockRUN();
    /* 初始化调试串口 */
        BOARD_InitDebugConsole();
    /* 打印系统时钟 */
        PRINTF("\r\n");
        PRINTF("*****欢迎使用野火i.MX RT1052 开发板*****\r\n");
        PRINTF("CPU:             %d Hz\r\n", CLOCK_GetFreq(kCLOCK_CpuClk));
        PRINTF("AHB:             %d Hz\r\n", CLOCK_GetFreq(kCLOCK_AhbClk));
        PRINTF("SEMC:            %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SemcClk));
        PRINTF("SYSPLL:          %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllClk));
    PRINTF("SYSPLLPFD0:      %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllPfd0Clk));
    PRINTF("SYSPLLPFD1:      %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllPfd1Clk));
    PRINTF("SYSPLLPFD2:      %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllPfd2Clk));
    PRINTF("SYSPLLPFD3:      %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllPfd3Clk));
    
    /* 初始化SysTick */
        SysTick_Config(SystemCoreClock / configTICK_RATE_HZ);
    
    /* 硬件BSP初始化统统放在这里，比如LED，串口，LCD等 */
    
    /* LED 端口初始化 */
        LED_GPIO_Config();
    
    
    /* KEY 端口初始化 */
        Key_GPIO_Config();
    
    }
    }

执行到BSP_Init()函数的时候，操作系统完全都还没有涉及到，即BSP_Init()函数所做的工作跟我们以前编写的裸机工程里面的硬件初始化工作是一模一样的。运行完BSP_Init ()函数，接下来才慢慢启动操作系统，最后运行创建好的任务。有时候任务创建好，整个系统跑起来了，可想要的实验现象就是出
不来，比如LED不会亮，串口没有输出，LCD没有显示等等。如果是初学者，这个时候就会心急如焚，四处求救，那怎么办？这个时候如何排除是硬件的问题还是系统的问题，这里面有个小小的技巧，即在硬件初始化好之后，顺便测试下硬件，测试方法跟裸机编程一样，具体实现见 代码清单14_2_ 的高亮部分。

.. code-block:: c
    :caption: 代码清单:14_2 BSP_Init()中添加硬件测试函数
    :name: 代码清单14_2
    :linenos:

    /***********************************************************************
    * @ 函数名： BSP_Init
    * @ 功能说明：
    板级外设初始化，所有板子上的初始化均可放在这个函数里面
    * @ 参数：
    * @ 返回值：无
    *********************************************************************/
    static void BSP_Init(void)
    {
    /* 初始化内存保护单元 */
        BOARD_ConfigMPU();
    /* 初始化开发板引脚 */
        BOARD_InitPins();
    /* 初始化开发板时钟 */
        BOARD_BootClockRUN();
    /* 初始化调试串口 */
        BOARD_InitDebugConsole();
    /* 此处省略打印系统时钟 */
    
    /* 初始化SysTick */
        SysTick_Config(SystemCoreClock / configTICK_RATE_HZ);
    
    /* 硬件BSP初始化统统放在这里，比如LED，串口，LCD等 */
    
    /* LED 端口初始化 */
        LED_GPIO_Config();						(1)
    /*测试硬件是否正常*/
        CORE_BOARD_LED_ON						(2)
    
    /* 让程序停在这里，不再继续往下执行 */
    while (1);						(3)
    
    /* KEY 端口初始化 */
        Key_GPIO_Config();
    
    }


代码清单14_2_ **(1)**\ ：初始化硬件后，顺便测试硬件，看下硬件是否正常工作。

代码清单14_2_ **(2)**\ ：可以继续添加其它的硬件初始化和测试。硬件确认没有问题之后，硬件测试代码可删可不删，因为BSP_Init()函数只执行一遍。

代码清单14_2_ **(3)**\ ：方便测试硬件好坏，让程序停在这里，不再继续往下执行，当测试完毕后，这个while(1);必须删除。

创建单任务—SRAM静态内存
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

这里，我们创建一个单任务，任务使用的栈和任务控制块都使用静态内存，即预先定义好的全局变量，这些预先定义好的全局变量都存在内部的SRAM中。

定义任务函数
^^^^^^^^^^^^

任务实际上就是一个无限循环且不带返回值的C函数。目前，我们创建一个这样的任务，让开发板上面的LED灯以500ms的频率闪烁，具体实现见 代码清单14_3_ 。


.. code-block:: c
    :caption: 代码清单:14_3 定义任务函数
    :name: 代码清单14_3
    :linenos:

     static voidLED_Task (void* parameter)
    {
    while (1)					(1)
        {
            LED1_ON;
    vTaskDelay(500);   /* 延时500个tick */(2)
    
            LED1_OFF;
    vTaskDelay(500);   /* 延时500个tick */
    
        }
    }


代码清单14_3_ **(1)**\ ：任务必须是一个死循环，否则任务将通过LR返回，如果LR指向了非法的内存就会产生HardFault_Handler，而FreeRTOS指向一个死循环，那么任务返回之后就在死循环中执行，这样子的任务是不安全的，所以避免这种情况，任务一般都是死循环并且无返回值的。我们的AppTaskC
reate任务，执行一次之后就进行删除，则不影响系统运行，所以，只执行一次的任务在执行完毕要记得及时删除。

代码清单14_3_ **(2)**\ ：任务里面的延时函数必须使用FreeRTOS里面提供的延时函数，并不能使用我们裸机编程中的那种延时。这两种的延时的区别是FreeRTOS里面的延时是阻塞延时，即调用vTaskDelay()函数的时候，当前任务会被挂起，调度器会切换到其它就绪的任务，从而实现多任务。如果还是使用裸机编
程中的那种延时，那么整个任务就成为了一个死循环，如果恰好该任务的优先级是最高的，那么系统永远都是在这个任务中运行，比它优先级更低的任务无法运行，根本无法实现多任务。

空闲任务与定时器任务堆栈函数实现
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

当我们使用了静态创建任务的时候，configSUPPORT_STATIC_ALLOCATION这个宏定义必须为1（在FreeRTOSConfig.h文件中），并且我们需要实现两个函数：vApplicationGetIdleTaskMemory()与vApplicationGetTimerTaskMe
mory()，这两个函数是用户设定的空闲（Idle）任务与定时器（Timer）任务的堆栈大小，必须由用户自己分配，而不能是动态分配，具体见 代码清单14_4_ 高亮部分。


.. code-block:: c
    :caption: 代码清单:14_4 空闲任务与定时器任务堆栈函数实现
    :emphasize-lines: 2, 4, 7, 9, 22-29, 42-48
    :name: 代码清单14_4
    :linenos:

    /* 空闲任务任务堆栈 */
    static StackType_t Idle_Task_Stack[configMINIMAL_STACK_SIZE];
    /* 定时器任务堆栈 */
    static StackType_t Timer_Task_Stack[configTIMER_TASK_STACK_DEPTH];

    /* 空闲任务控制块 */
    static StaticTask_t Idle_Task_TCB;
    /* 定时器任务控制块 */
    static StaticTask_t Timer_Task_TCB;

    /**
    *******************************************************************
    * @brief  获取空闲任务的任务堆栈和任务控制块内存
    *ppxTimerTaskTCBBuffer	:	任务控制块内存
    *ppxTimerTaskStackBuffer	:	任务堆栈内存
    *pulTimerTaskStackSize	:	任务堆栈大小
    * @author  fire
    * @version V1.0
    * @date    2018-xx-xx
    **********************************************************************
    */
    void vApplicationGetIdleTaskMemory(StaticTask_t **ppxIdleTaskTCBBuffer,
                                    StackType_t **ppxIdleTaskStackBuffer,
    uint32_t *pulIdleTaskStackSize)
    {
        *ppxIdleTaskTCBBuffer=&Idle_Task_TCB;/* 任务控制块内存 */
        *ppxIdleTaskStackBuffer=Idle_Task_Stack;/* 任务堆栈内存 */
        *pulIdleTaskStackSize=configMINIMAL_STACK_SIZE;/* 任务堆栈大小 */
    }

    /**
    *********************************************************************
    * @brief  获取定时器任务的任务堆栈和任务控制块内存
        *ppxTimerTaskTCBBuffer	:	任务控制块内存
        *ppxTimerTaskStackBuffer	:	任务堆栈内存
        *pulTimerTaskStackSize	:	任务堆栈大小
    * @author  fire
    * @version V1.0
    * @date    2018-xx-xx
    **********************************************************************
    */
    void vApplicationGetTimerTaskMemory(StaticTask_t **ppxTimerTaskTCBBuffer,
                                        StackType_t **ppxTimerTaskStackBuffer,
    uint32_t *pulTimerTaskStackSize)
    {
        *ppxTimerTaskTCBBuffer=&Timer_Task_TCB;/* 任务控制块内存 */
        *ppxTimerTaskStackBuffer=Timer_Task_Stack;/* 任务堆栈内存 */
        *pulTimerTaskStackSize=configTIMER_TASK_STACK_DEPTH;/* 任务堆栈大小 */
    }



定义任务栈
^^^^^^^^^^

目前我们只创建了一个任务，当任务进入延时的时候，因为没有另外就绪的用户任务，那么系统就会进入空闲任务，空闲任务是FreeRTOS系统自己启动的一个任务，优先级最低。当整个系统都没有就绪任务的时候，系统必须保证有一个任务在运行，空闲任务就是为这个设计的。当用户任务延时到期，又会从空闲任务切换回用户任务
。

在FreeRTOS系统中，每一个任务都是独立的，他们的运行环境都单独的保存在他们的栈空间当中。那么在定义好任务函数之后，我们还要为任务定义一个栈，目前我们使用的是静态内存，所以任务栈是一个独立的全局变量，具体见 代码清单14_5_ 。任务的栈占用的是MCU内部的RAM，当任务越多的时候，需要使用的栈空间就
越大，即需要使用的RAM空间就越多。一个MCU能够支持多少任务，就得看你的RAM空间有多少。

代码清单‑5定义任务栈

.. code-block:: c
    :caption: 代码清单:14_5 定义任务栈
    :name: 代码清单14_5
    :linenos:

    /* AppTaskCreate任务任务堆栈 */
    static StackType_t AppTaskCreate_Stack[128];
    
    /* LED任务堆栈 */
    static StackType_t LED_Task_Stack[128];


在大多数系统中需要做栈空间地址对齐，在FreeRTOS中是以8字节大小对齐，并且会检查堆栈是否已经对齐，其中portBYTE_ALIGNMENT是在portmacro.h里面定义的一个宏，其值为8，就是配置为按8字节对齐，当然用户可以选择按1、2、4、8、16、32等字节对齐，目前默认为8，具体见 代码清单14_6_ 。

.. code-block:: c
    :caption: 代码清单:14_6 栈空间地址对齐实现
    :name: 代码清单14_6
    :linenos:

    #define portBYTE_ALIGNMENT			8
 
    #if portBYTE_ALIGNMENT == 8
    #define portBYTE_ALIGNMENT_MASK ( 0x0007 )
    #endif
    
    pxTopOfStack = pxNewTCB->pxStack + ( ulStackDepth - ( uint32_t ) 1 );
    pxTopOfStack = ( StackType_t * ) ( ( ( portPOINTER_SIZE_TYPE ) pxTopOfStack ) &
                        ( ~( ( portPOINTER_SIZE_TYPE ) portBYTE_ALIGNMENT_MASK ) ) );
    
    /* 检查计算出的堆栈顶部的对齐方式是否正确。 */
    configASSERT( ( ( ( portPOINTER_SIZE_TYPE ) pxTopOfStack &
            ( portPOINTER_SIZE_TYPE ) portBYTE_ALIGNMENT_MASK ) == 0UL ) );


定义任务控制块
^^^^^^^^^^^^^^

定义好任务函数和任务栈之后，我们还需要为任务定义一个任务控制块，通常我们称这个任务控制块为任务的身份证。在C代码上，任务控制块就是一个结构体，里面有非常多的成员，这些成员共同描述了任务的全部信息，具体见 代码清单14_7_。


.. code-block:: c
    :caption: 代码清单:14_7 定义任务控制块
    :name: 代码清单14_7
    :linenos:

     /* AppTaskCreate 任务控制块 */
    static StaticTask_t AppTaskCreate_TCB;
    /* AppTaskCreate 任务控制块 */
    static StaticTask_t LED_Task_TCB;


静态创建任务
^^^^^^^^^^^^

一个任务的三要素是任务主体函数，任务栈，任务控制块，那么怎么样把这三个要素联合在一起？FreeRTOS里面有一个叫静态任务创建函数xTaskCreateStatic()，它就是干这个活的。它将任务主体函数，任务栈（静态的）和任务控制块（静态的）这三者联系在一起，让任务可以随时被系统启动，具体见 
代码清单14_8_ 。

代码清单‑8静态创建任务

.. code-block:: c
    :caption: 代码清单:14_8 定义任务控制块
    :name: 代码清单14_8
    :linenos:

    /* 创建 AppTaskCreate 任务 */
    AppTaskCreate_Handle = xTaskCreateStatic((TaskFunction_t)AppTaskCreate, //任务函数(1)
                           (const char* 	)"AppTaskCreate",//任务名称(2)
                       (uint32_t	)128,	//任务堆栈大小	(3)
                       (void* 		)NULL,	//传递给任务函数的参数(4)
                       (UBaseType_t 	)3, 	//任务优先级	(5)
                       (StackType_t*   )AppTaskCreate_Stack, //任务堆栈(6)
                       (StaticTask_t*  )&AppTaskCreate_TCB); //任务控制块(7)

    if (NULL != AppTaskCreate_Handle) /* 创建成功 */
        vTaskStartScheduler();   /* 启动任务，开启调度 */




代码清单14_8_ **(1)**\ ：任务入口函数，即任务函数的名称，需要我们自己定义并且实现。

代码清单14_8_ **(2)**\ ：任务名字，字符串形式，最大长度由FreeRTOSConfig.h中定义的configMAX_TASK_NAME_LEN宏指定，多余部分会被自动截掉，这里任务名字最好要与任务函数入口名字一致，方便进行调试。

代码清单14_8_ **(3)**\ ：任务堆栈大小，单位为字，在32位的处理器下（I.MX RT），一个字等于4个字节，那么任务大小就为128 \* 4字节。

代码清单14_8_ **(4)**\ ：任务入口函数形参，不用的时候配置为0或者NULL即可。

代码清单14_8_ **(5)**\ ：任务的优先级。优先级范围根据FreeRTOSConfig.h中的宏configMAX_PRIORITIES决定，如果使能configUSE_PORT_OPTIMISED_TASK_SELECTION，这个宏定义，则最多支持32个优先级；如果不用特殊方法查找下一个运行的任务，那么则
不强制要求限制最大可用优先级数目。在FreeRTOS中，数值越大优先级越高，0代表最低优先级。

代码清单14_8_ **(6)**\ ：任务栈起始地址，只有在使用静态内存的时候才需要提供，在使用动态内存的时候会根据提供的任务栈大小自动创建。

代码清单14_8_ **(7)**\
：任务控制块指针，在使用静态内存的时候，需要给任务初始化函数xTaskCreateStatic()传递预先定义好的任务控制块的指针。在使用动态内存的时候，任务创建函数xTaskCreate()会返回一个指针指向任务控制块，该任务控制块是xTaskCreate()函数里面动态分配的一块内存。

启动任务
^^^^^^^^

当任务创建好后，是处于任务就绪（Ready），在就绪态的任务可以参与操作系统的调度。但是此时任务仅仅是创建了，还未开启任务调度器，也没创建空闲任务与定时器任务（如果使能了configUSE_TIMERS这个宏定义），那这两个任务就是在启动任务调度器中实现，每个操作系统，任务调度器只启动一次，之后就不
会再次执行了，FreeRTOS中启动任务调度器的函数是vTaskStartScheduler()，并且启动任务调度器的时候就不会返回，从此任务管理都由FreeRTOS管理，此时才是真正进入实时操作系统中的第一步，具体见 代码清单14_9_ 。

.. code-block:: c
    :caption: 代码清单:14_9 启动任务
    :name: 代码清单14_9
    :linenos:

    /* 启动任务，开启调度 */
    vTaskStartScheduler();   


main.c文件内容全貌
^^^^^^^^^^^^^^^^^^^^^^^^

现在我们把任务主体，任务栈，任务控制块这三部分代码统一放到main.c中，我们在main.c文件中创建一个AppTaskCreate任务，这个任务是用于创建用户任务，为了方便管理，我们的所有的任务创建都统一放在这个函数中，
在这个函数中创建成功的任务就可以直接参与任务调度了，具体内容见 代码清单14_10_ 。

代码清单‑10 main.c文件内容全貌

.. code-block:: c
    :caption: 代码清单:14_10 main.c文件内容全貌
    :name: 代码清单14_10
    :linenos:

    /**
    ******************************************************************
    * @file    main.c
    * @author  fire
    * @version V1.0
    * @date    2018-xx-xx
    * @brief   SRAM动态创建单任务
    ******************************************************************
    * @attention
    *
    * 实验平台:野火  i.MXRT1052开发板
    * 论坛    :http://www.firebbs.cn
    * 淘宝    :http://firestm32.taobao.com
    *
    ******************************************************************
    */
        #include"fsl_debug_console.h"
        
        #include"board.h"
        #include"pin_mux.h"
        #include"clock_config.h"
        
        #include"./led/bsp_led.h"
        #include"./key/bsp_key.h"
        
        /* FreeRTOS头文件 */
        #include"FreeRTOS.h"
        #include"task.h"
        
        /**************************** 任务句柄 ********************************/
        /*
        * 任务句柄是一个指针，用于指向一个任务，当任务创建好之后，它就具有了一个任务句柄
        * 以后我们要想操作这个任务都需要通过这个任务句柄，如果是自身的任务操作自己，那么
        * 这个句柄可以为NULL。
        */
        /* 创建任务句柄 */
        static TaskHandle_t AppTaskCreate_Handle;
        /* LED任务句柄 */
        static TaskHandle_t LED_Task_Handle;
        
        /********************************** 内核对象句柄 
        *******************************/
        /*
        * 信号量，消息队列，事件标志组，软件定时器这些都属于内核的对象，要想使用这些内核
        * 对象，必须先创建，创建成功之后会返回一个相应的句柄。实际上就是一个指针，后续我
        * 们就可以通过这个句柄操作这些内核对象。
        *
        * 内核对象说白了就是一种全局的数据结构，通过这些数据结构我们可以实现任务间的通信，
        * 任务间的事件同步等各种功能。至于这些功能的实现我们是通过调用这些内核对象的函数
        * 来完成的
        *
        */


        /******************************* 全局变量声明 
        *********************************/
        /*
        * 当我们在写应用程序的时候，可能需要用到一些全局变量。
        */
        /* AppTaskCreate任务任务堆栈 */
        static StackType_t AppTaskCreate_Stack[128];
        /* LED任务堆栈 */
        static StackType_t LED_Task_Stack[128];

        /* AppTaskCreate 任务控制块 */
        static StaticTask_t AppTaskCreate_TCB;
        /* AppTaskCreate 任务控制块 */
        static StaticTask_t LED_Task_TCB;

        /* 空闲任务任务堆栈 */
        static StackType_t Idle_Task_Stack[configMINIMAL_STACK_SIZE];
        /* 定时器任务堆栈 */
        static StackType_t Timer_Task_Stack[configTIMER_TASK_STACK_DEPTH];

        /* 空闲任务控制块 */
        static StaticTask_t Idle_Task_TCB;
        /* 定时器任务控制块 */
        static StaticTask_t Timer_Task_TCB;

        /*
        *************************************************************************
        *                             函数声明
        *************************************************************************
        */
        static void AppTaskCreate(void);/* 用于创建任务 */

        static void LED_Task(void* pvParameters);/* LED_Task任务实现 */

        static void BSP_Init(void);/* 用于初始化板载相关资源 */

        /**
        * 使用了静态分配内存，以下这两个函数是由用户实现，函数在task.c文件中有引用
        * 当且仅当 configSUPPORT_STATIC_ALLOCATION 这个宏定义为 1 的时候才有效
        */
        void vApplicationGetTimerTaskMemory(StaticTask_t **ppxTimerTaskTCBBuffer,
                                            StackType_t **ppxTimerTaskStackBuffer,
        uint32_t *pulTimerTaskStackSize);

        void vApplicationGetIdleTaskMemory(StaticTask_t **ppxIdleTaskTCBBuffer,
                                        StackType_t **ppxIdleTaskStackBuffer,
        uint32_t *pulIdleTaskStackSize);
        
        /*****************************************************************
        * @brief  主函数
        * @param  无
        * @retval 无
        * @note   第一步：开发板硬件初始化
        第二步：创建APP应用任务
        第三步：启动FreeRTOS，开始多任务调度
        ****************************************************************/
        int main(void)
        {
        /* 开发板硬件初始化 */
            BSP_Init();

            PRINTF("这是一个[野火]-全系列开发板-FreeRTOS-静态创建单任务!\r\n");
        /* 创建 AppTaskCreate 任务 */
            AppTaskCreate_Handle = 
        xTaskCreateStatic((TaskFunction_t  )AppTaskCreate,   //任务函数
                                (const char*   )"AppTaskCreate",   //任务名称
                                (uint32_t    )128, //任务堆栈大小
                                (void*         )NULL,        //传递给任务函数的参数
                                (UBaseType_t   )3,   //任务优先级
                                (StackType_t*   )AppTaskCreate_Stack,  //任务堆栈
                                (StaticTask_t*  )&AppTaskCreate_TCB);  //任务控制块

        if (NULL != AppTaskCreate_Handle) /* 创建成功 */
                vTaskStartScheduler();   /* 启动任务，开启调度 */

        while (1);  /* 正常不会执行到这里 */
        }


        /***********************************************************************
        * @ 函数名： AppTaskCreate
        * @ 功能说明：为了方便管理，所有的任务创建函数都放在这个函数里面
        * @ 参数：无
        * @ 返回值：无
        
        ******************************************************************/
        static void AppTaskCreate(void)
        {
            taskENTER_CRITICAL();           //进入临界区

        /* 创建LED_Task任务 */
        LED_Task_Handle = xTaskCreateStatic((TaskFunction_t )LED_Task,    //任务函数
                                            (const char*  )"LED_Task",    //任务名称
                                            (uint32_t     )128,         //任务堆栈大小
                                            (void*        )NULL,        //传递给任务函数

                                            (UBaseType_t  )4,         //任务优先级
                                            (StackType_t*   )LED_Task_Stack,  //任务堆

                                            (StaticTask_t*  )&LED_Task_TCB);  //任务控


        if (NULL != LED_Task_Handle) /* 创建成功 */
                PRINTF("LED_Task任务创建成功!\n");
        else
                PRINTF("LED_Task任务创建失败!\n");

            vTaskDelete(AppTaskCreate_Handle); //删除AppTaskCreate任务

            taskEXIT_CRITICAL();            //退出临界区
        }



        /**********************************************************************
        * @ 函数名： LED_Task
        * @ 功能说明： LED_Task任务主体
        * @ 参数：
        * @ 返回值：无
    ********************************************************************/
        static void LED_Task(void* parameter)
        {
        while (1) {
                LED1_ON;
                vTaskDelay(500);   /* 延时500个tick */
                PRINTF("LED_Task Running,LED1_ON\r\n");

                LED1_OFF;
                vTaskDelay(500);   /* 延时500个tick */
                PRINTF("LED_Task Running,LED1_OFF\r\n");
            }
        }


        /**
        **********************************************************************
        * @brief  获取空闲任务的任务堆栈和任务控制块内存
        *         ppxTimerTaskTCBBuffer :   任务控制块内存
        *         ppxTimerTaskStackBuffer : 任务堆栈内存
        *         pulTimerTaskStackSize :   任务堆栈大小
        * @author  fire
        * @version V1.0
        * @date    2018-xx-xx
        **********************************************************************
        */
        void vApplicationGetIdleTaskMemory(StaticTask_t **ppxIdleTaskTCBBuffer,
                                        StackType_t **ppxIdleTaskStackBuffer,
        uint32_t *pulIdleTaskStackSize)
        {
            *ppxIdleTaskTCBBuffer=&Idle_Task_TCB;/* 任务控制块内存 */
            *ppxIdleTaskStackBuffer=Idle_Task_Stack;/* 任务堆栈内存 */
            *pulIdleTaskStackSize=configMINIMAL_STACK_SIZE;/* 任务堆栈大小 */
        }

        /**
        *********************************************************************
        * @brief  获取定时器任务的任务堆栈和任务控制块内存
        *         ppxTimerTaskTCBBuffer :   任务控制块内存
        *         ppxTimerTaskStackBuffer : 任务堆栈内存
        *         pulTimerTaskStackSize :   任务堆栈大小
        * @author  fire
        * @version V1.0
        * @date    2018-xx-xx
        **********************************************************************
        */
        oid vApplicationGetTimerTaskMemory(StaticTask_t **ppxTimerTaskTCBBuffer,
                                        StackType_t **ppxTimerTaskStackBuffer,
        uint32_t *pulTimerTaskStackSize)
        {
            *ppxTimerTaskTCBBuffer=&Timer_Task_TCB;/* 任务控制块内存 */
            *ppxTimerTaskStackBuffer=Timer_Task_Stack;/* 任务堆栈内存 */
            *pulTimerTaskStackSize=configTIMER_TASK_STACK_DEPTH;/* 任务堆栈大小 */
        }
        /***********************************************************************
        * @ 函数名： BSP_Init
        * @ 功能说明：板级外设初始化，所有板子上的初始化均可放在这个函数里面
        * @ 参数：
        * @ 返回值：无
        *********************************************************************/
        static void BSP_Init(void)
        {
        /* 初始化内存保护单元 */
            BOARD_ConfigMPU();
        /* 初始化开发板引脚 */
            BOARD_InitPins();
        /* 初始化开发板时钟 */
            BOARD_BootClockRUN();
        /* 初始化调试串口 */
            BOARD_InitDebugConsole();
        /* 打印系统时钟 */
            PRINTF("\r\n");
            PRINTF("*****欢迎使用野火i.MX RT1052 开发板*****\r\n");
            PRINTF("CPU:             %d Hz\r\n", CLOCK_GetFreq(kCLOCK_CpuClk));
            PRINTF("AHB:             %d Hz\r\n", CLOCK_GetFreq(kCLOCK_AhbClk));
            PRINTF("SEMC:            %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SemcClk));
            PRINTF("SYSPLL:          %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllClk));
            PRINTF("SYSPLLPFD0:      %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllPfd0Clk));
            PRINTF("SYSPLLPFD1:      %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllPfd1Clk));
            PRINTF("SYSPLLPFD2:      %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllPfd2Clk));
            PRINTF("SYSPLLPFD3:      %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllPfd3Clk));
        
        /* 初始化SysTick */
            SysTick_Config(SystemCoreClock / configTICK_RATE_HZ);
        
        /* 硬件BSP初始化统统放在这里，比如LED，串口，LCD等 */
        
        /* LED 端口初始化 */
            LED_GPIO_Config();
        
        /* KEY 端口初始化 */
            Key_GPIO_Config();
        
        }
        /****************************END OF FILE**********************/


注意：在使用静态创建任务的时候必须要将FreeRTOSConfig.h中的configSUPPORT_STATIC_ALLOCATION宏配置为1。

下载验证
~~~~~~~~

将程序编译好，用DAP仿真器把程序下载到野火I.MX RT系列开发板（具体型号根据你买的板子而定，每个型号的板子都配套有对应的程序），可以看到板子上面的LED灯已经在闪烁，说明我们创建的单任务（使用静态内存）已经跑起来了。

在当前这个例程，任务的栈，任务的控制块用的都是静态内存，必须由用户预先定义，这种方法我们在使用FreeRTOS的时候用的比较少，通常的方法是在任务创建的时候动态的分配任务栈和任务控制块的内存空间，接下来我们讲解下“创建单任务—SRAM动态内存”的方法。

创建单任务—SRAM动态内存
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

这里，我们创建一个单任务，任务使用的栈和任务控制块是在创建任务的时候FreeRTOS动态分配的，并不是预先定义好的全局变量。那这些动态的内存堆是从哪里来？继续往下看。

动态内存空间的堆从哪里来
^^^^^^^^^^^^^^^^^^^^^^^^

在创建单任务—SRAM静态内存的例程中，任务控制块和任务栈的内存空间都是从内部的SRAM里面分配的，具体分配到哪个地址由编译器决定。现在我们开始使用动态内存，即堆，其实堆也是内存，也属于SRAM。FreeRTOS做法是在SRAM里面定义一个大数组，也就是堆内存，供FreeRTOS的动态内存分配函数使
用，在第一次使用的时候，系统会将定义的堆内存进行初始化，这些代码在FreeRTOS提供的内存管理方案中实现（heap_1.c、heap_2.c、heap_4.c等，具体的内存管理方案后面详细讲解），具体见 代码清单14_11_ 。



.. code-block:: c
    :caption: 代码清单:14_11 定义FreeRTOS的堆到内部SRAM
    :name: 代码清单14_11
    :linenos:

    //系统所有总的堆大小
    #define configTOTAL_HEAP_SIZE		 ((size_t)(36*1024))	(1)
    static uint8_t ucHeap[ configTOTAL_HEAP_SIZE ];			(2)	
    /* 如果这是第一次调用malloc那么堆将需要
    初始化，以设置空闲块列表。*/
    if ( pxEnd == NULL )	
    {
        prvHeapInit();						(3)	
    } else
    {
        mtCOVERAGE_TEST_MARKER();
    }


代码清单14_11_ **(1)**\ ：堆内存的大小为configTOTAL_HEAP_SIZE，在FreeRTOSConfig.h中由我们自己定义，configSUPPORT_DYNAMIC_ALLOCATION 这个宏定义在使用FreeRTOS操作系统的时候必须开启。

代码清单14_11_ **(2)**\ ：从内部SRAMM里面定义一个静态数组ucHeap，大小由configTOTAL_HEAP_SIZE这个宏决定，目前定义为36KB。定义的堆大小不能超过内部SRAM的总大小。

代码清单14_11_ **(3)**\ ：如果这是第一次调用malloc那么需要将堆进行初始化，以设置空闲块列表，方便以后分配内存，初始化完成之后会取得堆的结束地址，在MemMang中的5个内存分配heap_x.c文件中实现。


定义任务函数
^^^^^^^^^^^^

使用动态内存的时候，任务的主体函数与使用静态内存时是一样的，具体见 代码清单14_12_。

.. code-block:: c
    :caption: 代码清单:14_12 定义任务函数
    :name: 代码清单14_12
    :linenos:

     static void LED_Task (void* parameter)
    {
    while (1)					(1)		
        {
            LED1_ON;
    vTaskDelay(500);   /* 延时500个tick */(2)
    
            LED1_OFF;
    vTaskDelay(500);   /* 延时500个tick */
        }
    }

代码清单14_12_ **(1)**\ ：任务必须是一个死循环，否则任务将通过LR返回，如果LR指向了非法的内存就会产生HardFault_Handler，而FreeRTOS指向一个任务退出函数prvTaskExitError()，里面是一个死循环，那么任务返回之后就在死循环中执行，这样子的任务是不
安全的，所以避免这种情况，任务一般都是死循环并且无返回值的。我们的AppTaskCreate任务，执行一次之后就进行删除，则不影响系统运行，所以，只执行一次的任务在执行完毕要记得及时删除。

代码清单14_12_ **(2)**\ ：任务里面的延时函数必须使用FreeRTOS里面提供的延时函数，并不能使用我们裸机编程中的那种延时。这两种的延时的区别是FreeRTOS里面的延时是阻塞延时，即调用vTaskDelay()函数的时候，当前任务会被挂起，调度器会切换到其它就绪的任务，从而实现多任
务。如果还是使用裸机编程中的那种延时，那么整个任务就成为了一个死循环，如果恰好该任务的优先级是最高的，那么系统永远都是在这个任务中运行，比它优先级更低的任务无法运行，根本无法实现多任务。


定义任务栈
^^^^^^^^^^

使用动态内存的时候，任务栈在任务创建的时候创建，不用跟使用静态内存那样要预先定义好一个全局的静态的栈空间，动态内存就是按需分配内存，随用随取。

定义任务控制块指针
^^^^^^^^^^^^^^^^^^

使用动态内存时候，不用跟使用静态内存那样要预先定义好一个全局的静态的任务控制块空间。任务控制块是在任务创建的时候分配内存空间创建，
任务创建函数会返回一个指针，用于指向任务控制块，所以要预先为任务栈定义一个任务控制块指针，也是我们常说的任务句柄，具体见 代码清单14_13_ 。



.. code-block:: c
    :caption: 代码清单:14_13 定义任务句柄
    :name: 代码清单14_13_
    :linenos:

    /**************************** 任务句柄 ********************************/
    /*
    * 任务句柄是一个指针，用于指向一个任务，当任务创建好之后，它就具有了一个任务句柄
    * 以后我们要想操作这个任务都需要通过这个任务句柄，如果是自身的任务操作自己，那么
    * 这个句柄可以为NULL。
    */
    /* 创建任务句柄 */
    static TaskHandle_t AppTaskCreate_Handle = NULL;
    /* LED任务句柄 */
    static TaskHandle_t LED_Task_Handle = NULL;


动态创建任务
^^^^^^^^^^^^

使用静态内存时，使用xTaskCreateStatic()来创建一个任务，而使用动态内存的时，则使用xTaskCreate()函数来创建一个任务，
两者的函数名不一样，具体的形参也有区别，具体见 代码清单14_14_。



.. code-block:: c
    :caption: 代码清单:14_14 动态创建任务
    :name: 代码清单14_14_
    :linenos:

    /* 创建AppTaskCreate任务 */
    xReturn = xTaskCreate((TaskFunction_t )AppTaskCreate,  /* 任务入口函数 */(1)
                        (const char*    )"AppTaskCreate",/* 任务名字 */(2)
                        (uint16_t       )512,  /* 任务栈大小 */	(3)
                        (void*          )NULL,/* 任务入口函数参数 */	(4)
                        (UBaseType_t    )1, /* 任务的优先级 */	(5)
                        (TaskHandle_t*  )&AppTaskCreate_Handle);/* 任务控制块指针 */(6)
    /* 启动任务调度 */
    if (pdPASS == xReturn)
        vTaskStartScheduler();   /* 启动任务，开启调度 */


代码清单14_14_ **(1)**\ ：任务入口函数，即任务函数的名称，需要我们自己定义并且实现。

代码清单14_14_ **(2)**\ ：任务名字，字符串形式，最大长度由FreeRTOSConfig.h中定义的configMAX_TASK_NAME_LEN宏指定，多余部分会被自动截掉，这里任务名字最好要与任务函数入口名字一致，方便进行调试。

代码清单14_14_ **(3)**\ ：任务堆栈大小，单位为字，在32位的处理器下（I.MX RT），一个字等于4个字节，那么任务大小就为128 \* 4字节。

代码清单14_14_ **(4)**\ ：任务入口函数形参，不用的时候配置为0或者NULL即可。

代码清单14_14_ **(5)**\ ：任务的优先级。优先级范围根据FreeRTOSConfig.h中的宏configMAX_PRIORITIES决定，如果使能configUSE_PORT_OPTIMISED_TASK_SELECTION，这个宏定义，则最多支持32个优先级；如果不用特殊方法查找下
一个运行的任务，那么则不强制要求限制最大可用优先级数目。在FreeRTOS中，数值越大优先级越高，0代表最低优先级。

代码清单14_14_ **(6)**\
：任务控制块指针，在使用内存的时候，需要给任务初始化函数xTaskCreateStatic()传递预先定义好的任务控制块的指针。在使用动态内存的时候，任务创建函数xTaskCreate()会返回一个指针指向任务控制块，该任务控制块是xTaskCreate()函数里面动态分配的一块内存。


启动任务
^^^^^^^^

当任务创建好后，是处于任务就绪（Ready），在就绪态的任务可以参与操作系统的调度。但是此时任务仅仅是创建了，还未开启任务调度器，也没创建空闲任务与定时器任务（如果使能了configUSE_TIMERS这个宏定义），那这两个任务就是在启动任务调度器中实现，每个操作系统，任务调度器只启动一次，之后就不
会再次执行了，FreeRTOS中启动任务调度器的函数是vTaskStartScheduler()，并且启动任务调度器的时候就不会返回，从此任务管理都由FreeRTOS管理，此时才是真正进入实时操作系统中的第一步，具体见 代码清单14_15_ 。

.. code-block:: c
    :caption: 代码清单:14_15 启动任务
    :name: 代码清单14_15_
    :linenos:

    /* 启动任务调度 */
    if (pdPASS == xReturn)
        vTaskStartScheduler();   /* 启动任务，开启调度 */
    else
    return -1;

main.c文件内容全貌
^^^^^^^^^^^^^^^^^^^^^^^^

现在我们把任务主体，任务栈，任务控制块这三部分代码统一放到main.c中，我们在main.c文件中创建一个AppTaskCreate任务，这个任务是用于创建用户任务，
为了方便管理，我们的所有的任务创建都统一放在这个函数中，在这个函数中创建成功的任务就可以直接参与任务调度了，具体内容见 代码清单14_16_ 。

代码清单‑16main.c文件内容全貌

.. code-block:: c
    :caption: 代码清单:14_16 main.c文件内容全貌
    :name: 代码清单14_16_
    :linenos:

    /**
    ******************************************************************
    * @file    main.c
    * @author  fire
    * @version V1.0
    * @date    2018-xx-xx
    * @brief   SRAM动态创建单任务
    ******************************************************************
    * @attention
    *
    * 实验平台:野火  i.MXRT1052开发板
    * 论坛    :http://www.firebbs.cn
    * 淘宝    :http://firestm32.taobao.com
    *
    ******************************************************************
    */
        #include"fsl_debug_console.h"
        
        #include"board.h"
        #include"pin_mux.h"
        #include"clock_config.h"
        
        #include"./led/bsp_led.h"
        #include"./key/bsp_key.h"
        
        /* FreeRTOS头文件 */
        #include"FreeRTOS.h"
        #include"task.h"
        
        /**************************** 任务句柄 ********************************/
        /*
        * 任务句柄是一个指针，用于指向一个任务，当任务创建好之后，它就具有了一个任务句柄
        * 以后我们要想操作这个任务都需要通过这个任务句柄，如果是自身的任务操作自己，那么
        * 这个句柄可以为NULL。
        */
        /* 创建任务句柄 */
        static TaskHandle_t AppTaskCreate_Handle = NULL;
        /* LED任务句柄 */
        static TaskHandle_t LED_Task_Handle = NULL;
        
        /********************************** 内核对象句柄 
        *******************************/
        /*
        * 信号量，消息队列，事件标志组，软件定时器这些都属于内核的对象，要想使用这些内核
        * 对象，必须先创建，创建成功之后会返回一个相应的句柄。实际上就是一个指针，后续我
        * 们就可以通过这个句柄操作这些内核对象。
        *
        * 内核对象说白了就是一种全局的数据结构，通过这些数据结构我们可以实现任务间的通信，
        * 任务间的事件同步等各种功能。至于这些功能的实现我们是通过调用这些内核对象的函数
         * 来完成的
        *
        */


        /******************************* 全局变量声明 
        *********************************/
        /*
        * 当我们在写应用程序的时候，可能需要用到一些全局变量。
        */


        /*
        *************************************************************************
        *                             函数声明
        *************************************************************************
        */
        static void AppTaskCreate(void);/* 用于创建任务 */

        static void LED_Task(void* pvParameters);/* LED_Task任务实现 */

        static void BSP_Init(void);/* 用于初始化板载相关资源 */

        /*****************************************************************
        * @brief  主函数
        * @param  无
        * @retval 无
        * @note   第一步：开发板硬件初始化
        第二步：创建APP应用任务
        第三步：启动FreeRTOS，开始多任务调度
        ****************************************************************/
        int main(void)
        {
            BaseType_t xReturn = pdPASS;/* 定义一个创建信息返回值，默认为pdPASS */

        /* 开发板硬件初始化 */
            BSP_Init();
            PRINTF("这是一个[野火]-全系列开发板-FreeRTOS-动态创建任务!\r\n");
        /* 创建AppTaskCreate任务 */
            xReturn = xTaskCreate((TaskFunction_t )AppTaskCreate,/* 任务入口函数 */
                                (const char*    )"AppTaskCreate",/* 任务名字 */
                                (uint16_t       )512,  /* 任务栈大小 */
                                (void*          )NULL,/* 任务入口函数参数 */
                                (UBaseType_t    )1, /* 任务的优先级 */
            (TaskHandle_t*  )&AppTaskCreate_Handle);/* 任务控制块指针 */
        /* 启动任务调度 */
        if (pdPASS == xReturn)
                vTaskStartScheduler();   /* 启动任务，开启调度 */
        else
        return -1;

        while (1);  /* 正常不会执行到这里 */
        }
        
        
        /***********************************************************************
        * @ 函数名： AppTaskCreate
        * @ 功能说明：为了方便管理，所有的任务创建函数都放在这个函数里面
        * @ 参数：无
        * @ 返回值：无
        *******************************************************************/
        static void AppTaskCreate(void)
        {
            BaseType_t xReturn = pdPASS;/* 定义一个创建信息返回值，默认为pdPASS */
        
            taskENTER_CRITICAL();           //进入临界区
        
        /* 创建LED_Task任务 */
            xReturn = xTaskCreate((TaskFunction_t )LED_Task, /* 任务入口函数 */
                                (const char*    )"LED_Task",/* 任务名字 */
                                (uint16_t       )512,   /* 任务栈大小 */
                                (void*          )NULL,  /* 任务入口函数参数 */
                                (UBaseType_t    )2,     /* 任务的优先级 */
                                (TaskHandle_t*  )&LED_Task_Handle);/*任务控制块
        */
        if (pdPASS == xReturn)
                PRINTF("创建LED_Task任务成功!\r\n");
        
            vTaskDelete(AppTaskCreate_Handle); //删除AppTaskCreate任务
        
            taskEXIT_CRITICAL();            //退出临界区
        }
        
        
        
        /**********************************************************************
        * @ 函数名： LED_Task
        * @ 功能说明： LED_Task任务主体
        * @ 参数：
        * @ 返回值：无
        ********************************************************************/
        static void LED_Task(void* parameter)
        {
        while (1) {
                LED1_ON;
                vTaskDelay(500);   /* 延时500个tick */
                PRINTF("LED_Task Running,LED1_ON\r\n");
        
                LED1_OFF;
                vTaskDelay(500);   /* 延时500个tick */
                PRINTF("LED_Task Running,LED1_OFF\r\n");
            }
        }
        
        /***********************************************************************
        * @ 函数名： BSP_Init
        * @ 功能说明：板级外设初始化，所有板子上的初始化均可放在这个函数里面
        * @ 参数：
        * @ 返回值：无
        *********************************************************************/
        static void BSP_Init(void)
        {
        /* 初始化内存保护单元 */
            BOARD_ConfigMPU();
        /* 初始化开发板引脚 */
            BOARD_InitPins();
        /* 初始化开发板时钟 */
            BOARD_BootClockRUN();
        /* 初始化调试串口 */
            BOARD_InitDebugConsole();
        /* 打印系统时钟 */
        PRINTF("\r\n");
        PRINTF("*****欢迎使用野火i.MX RT1052 开发板*****\r\n");
        PRINTF("CPU:             %d Hz\r\n", CLOCK_GetFreq(kCLOCK_CpuClk));
        PRINTF("AHB:             %d Hz\r\n", CLOCK_GetFreq(kCLOCK_AhbClk));
        PRINTF("SEMC:            %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SemcClk));
        PRINTF("SYSPLL:          %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllClk));
        PRINTF("SYSPLLPFD0:      %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllPfd0Clk));
        PRINTF("SYSPLLPFD1:      %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllPfd1Clk));
        PRINTF("SYSPLLPFD2:      %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllPfd2Clk));
        PRINTF("SYSPLLPFD3:      %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllPfd3Clk));

        /* 初始化SysTick */
            SysTick_Config(SystemCoreClock / configTICK_RATE_HZ);

        /* 硬件BSP初始化统统放在这里，比如LED，串口，LCD等 */

        /* LED 端口初始化 */
            LED_GPIO_Config();


        /* KEY 端口初始化 */
            Key_GPIO_Config();

        }
        /****************************END OF FILE**********************/



其实动态创建与静态创建的差别就是特别小，以后我们使用FreeRTOS除非是特别说明，否则我们都使用动态创建任务。


下载验证
~~~~~~~~

将程序编译好，用DAP仿真器把程序下载到野火I.MX RT系列开发板（具体型号根据你买的板子而定，每个型号的板子都配套有对应的程序），可以看到板子上面的LED灯已经在闪烁，说明我们创建的单任务（使用动态内存）已经跑起来了。在往后的实验中，我们创建内核对象均采用动态内存分配方案。

创建多任务—SRAM动态内存
~~~~~~~~~~~~~~~~~~~~~~~~~~

创建多任务只需要按照创建单任务的套路依葫芦画瓢即可，接下来我们创建两个任务，任务1让一个LED灯闪烁，任务2让另外一个LED闪烁，两个LED闪烁的频率不一样，具体实现见 代码清单14_17_ 的高亮部分，两个任务的优先级不一样。


.. code-block:: c
    :caption: 代码清单:14_17 创建多任务—SRAM动态内存
    :emphasize-lines: 36, 38, 40, 113-143
    :name: 代码清单14_17
    :linenos:

    /**
    ******************************************************************
    * @file    main.c
    * @author  fire
    * @version V1.0
    * @date    2018-xx-xx
    * @brief   SRAM动态创建多任务
    ******************************************************************
    * @attention
    *
    * 实验平台:野火  i.MXRT1052开发板
      * 论坛    :http://www.firebbs.cn
    * 淘宝    :http://firestm32.taobao.com
    *
    ******************************************************************
    */
    #include"fsl_debug_console.h"

    #include"board.h"
    #include"pin_mux.h"
    #include"clock_config.h"

    #include"./led/bsp_led.h"
    #include"./key/bsp_key.h"

    /* FreeRTOS头文件 */
    #include"FreeRTOS.h"
    #include"task.h"
    /**************************** 任务句柄 ********************************/
    /*
    * 任务句柄是一个指针，用于指向一个任务，当任务创建好之后，它就具有了一个任务句柄
    * 以后我们要想操作这个任务都需要通过这个任务句柄，如果是自身的任务操作自己，那么
    * 这个句柄可以为NULL。
    */
    /* 创建任务句柄 */
    static TaskHandle_t AppTaskCreate_Handle = NULL;
    /* LED1任务句柄 */
    static TaskHandle_t LED1_Task_Handle = NULL;
    /* LED2任务句柄 */
    static TaskHandle_t LED2_Task_Handle = NULL;
    /********************************** 内核对象句柄 
    ******************************/
    /*
    * 信号量，消息队列，事件标志组，软件定时器这些都属于内核的对象，要想使用这些内核
    * 对象，必须先创建，创建成功之后会返回一个相应的句柄。实际上就是一个指针，后续我
    * 们就可以通过这个句柄操作这些内核对象。
    *
    * 内核对象说白了就是一种全局的数据结构，通过这些数据结构我们可以实现任务间的通信，
    * 任务间的事件同步等各种功能。至于这些功能的实现我们是通过调用这些内核对象的函数
    * 来完成的
    *
    */


    /******************************* 全局变量声明 
    *********************************/
    /*
    * 当我们在写应用程序的时候，可能需要用到一些全局变量。
    */


    /*
    *************************************************************************
    *                             函数声明
    *************************************************************************
    */
    static void AppTaskCreate(void);/* 用于创建任务 */

    static void LED1_Task(void* pvParameters);/* LED1_Task任务实现 */
    static void LED2_Task(void* pvParameters);/* LED2_Task任务实现 */

    static void BSP_Init(void);/* 用于初始化板载相关资源 */
    
    /*****************************************************************
    * @brief  主函数
    * @param  无
    * @retval 无
    * @note   第一步：开发板硬件初始化
    第二步：创建APP应用任务
    第三步：启动FreeRTOS，开始多任务调度
    ****************************************************************/
    int main(void)
    {
        BaseType_t xReturn = pdPASS;/* 定义一个创建信息返回值，默认为pdPASS */

    /* 开发板硬件初始化 */
        BSP_Init();
        PRINTF("这是一个[野火]-全系列开发板-FreeRTOS-动态创建多任务实验!\r\n");
    /* 创建AppTaskCreate任务 */
        xReturn = xTaskCreate((TaskFunction_t )AppTaskCreate,  /* 任务入口函数 

                            (const char*    )"AppTaskCreate",/* 任务名字 */
                            (uint16_t       )512,  /* 任务栈大小 */
                            (void*          )NULL,/* 任务入口函数参数 */
                            (UBaseType_t    )1, /* 任务的优先级 */
                            (TaskHandle_t*  )&AppTaskCreate_Handle);/* 任务
    块指针 */
    /* 启动任务调度 */
    if (pdPASS == xReturn)
            vTaskStartScheduler();   /* 启动任务，开启调度 */
    else
    return -1;

    while (1);  /* 正常不会执行到这里 */
    }
    
    
    /***********************************************************************
    * @ 函数名： AppTaskCreate
    * @ 功能说明：为了方便管理，所有的任务创建函数都放在这个函数里面
    * @ 参数：无
    * @ 返回值：无
    
    *******************************************************************/
    static void AppTaskCreate(void)
    {
        BaseType_t xReturn = pdPASS;/* 定义一个创建信息返回值，默认为pdPASS */
    
        taskENTER_CRITICAL();           //进入临界区
    
    /* 创建LED_Task任务 */
        xReturn = xTaskCreate((TaskFunction_t )LED1_Task, /* 任务入口函数 */
                            (const char*    )"LED1_Task",/* 任务名字 */
                            (uint16_t       )512,   /* 任务栈大小 */
                            (void*          )NULL,  /* 任务入口函数参数 */
                            (UBaseType_t    )2,     /* 任务的优先级 */
                            (TaskHandle_t*  )&LED1_Task_Handle);/* 任务控制
    针 */
    if (pdPASS == xReturn)
            PRINTF("创建LED1_Task任务成功!\r\n");
    
    /* 创建LED_Task任务 */
         xReturn = xTaskCreate((TaskFunction_t )LED2_Task, /* 任务入口函数 */
                           (const char*    )"LED2_Task",/* 任务名字 */
                           (uint16_t       )512,   /* 任务栈大小 */
                           (void*          )NULL,  /* 任务入口函数参数 */
                           (UBaseType_t    )3,     /* 任务的优先级 */
                            (TaskHandle_t*  )&LED2_Task_Handle);/* 任务控制
    针 */
    if (pdPASS == xReturn)
            PRINTF("创建LED2_Task任务成功!\r\n");
    
        vTaskDelete(AppTaskCreate_Handle); //删除AppTaskCreate任务
    
        taskEXIT_CRITICAL();            //退出临界区
    }
    
    
    
    /**********************************************************************
    * @ 函数名： LED_Task
    * @ 功能说明： LED_Task任务主体
    * @ 参数：
    * @ 返回值：无
    ********************************************************************/
    static void LED1_Task(void* parameter)
    {
    while (1) {
            LED1_ON;
            vTaskDelay(500);   /* 延时500个tick */
            PRINTF("LED1_Task Running,LED1_ON\r\n");
    
            LED1_OFF;
            vTaskDelay(500);   /* 延时500个tick */
            PRINTF("LED1_Task Running,LED1_OFF\r\n");
        }
    }
    
    /**********************************************************************
    * @ 函数名： LED_Task
    * @ 功能说明： LED_Task任务主体
    * @ 参数：
    * @ 返回值：无
    ********************************************************************/
    static void LED2_Task(void* parameter)
    {
    while (1) {
            LED2_ON;
            vTaskDelay(500);   /* 延时500个tick */
            PRINTF("LED2_Task Running,LED2_ON\r\n");
    
            LED2_OFF;
            vTaskDelay(500);   /* 延时500个tick */
            PRINTF("LED2_Task Running,LED2_OFF\r\n");
        }
    }
    /***********************************************************************
    * @ 函数名： BSP_Init
    * @ 功能说明：板级外设初始化，所有板子上的初始化均可放在这个函数里面
    * @ 参数：
    * @ 返回值：无
    *********************************************************************/
    static void BSP_Init(void)
    {
    /* 初始化内存保护单元 */
        BOARD_ConfigMPU();
    /* 初始化开发板引脚 */
        BOARD_InitPins();
    /* 初始化开发板时钟 */
        BOARD_BootClockRUN();
    /* 初始化调试串口 */
        BOARD_InitDebugConsole();
    /* 打印系统时钟 */
        PRINTF("\r\n");
        PRINTF("*****欢迎使用野火i.MX RT1052 开发板*****\r\n");
        PRINTF("CPU:             %d Hz\r\n", CLOCK_GetFreq(kCLOCK_CpuClk));
        PRINTF("AHB:             %d Hz\r\n", CLOCK_GetFreq(kCLOCK_AhbClk));
        PRINTF("SEMC:            %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SemcClk));
        PRINTF("SYSPLL:          %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllClk));
    PRINTF("SYSPLLPFD0:      %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllPfd0Clk));
    PRINTF("SYSPLLPFD1:      %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllPfd1Clk));
    PRINTF("SYSPLLPFD2:      %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllPfd2Clk));
    PRINTF("SYSPLLPFD3:      %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllPfd3Clk));

    /* 初始化SysTick */
        SysTick_Config(SystemCoreClock / configTICK_RATE_HZ);

    /* 硬件BSP初始化统统放在这里，比如LED，串口，LCD等 */

    /* LED 端口初始化 */
        LED_GPIO_Config();


    /* KEY 端口初始化 */
        Key_GPIO_Config();

    }
    /****************************END OF FILE**********************/






目前多任务我们只创建了两个，如果要创建3个、4个甚至更多都是同样的套路，容易忽略的地方是任务栈的大小，每个任务的优先级。大的任务，栈空间要设置大一点，重要的任务优先级要设置的高一点。



下载验证
~~~~

将程序编译好，用DAP仿真器把程序下载到野火I.MX RT系列开发板（具体型号根据你买的板子而定，每个型号的板子都配套有对应的程序），可以看到板子上面的两个LED灯以不同的频率在闪烁，说明我们创建的多任务（使用动态内存）已经跑起来了。在往后的实验中，我们创建内核对象均采用动态内存分配方案。
