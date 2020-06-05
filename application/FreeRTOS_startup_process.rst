.. vim: syntax=rst

FreeRTOS的启动流程
====================

在目前的RTOS中，主要有两种比较流行的启动方式，暂时还没有看到第三种，接下来我将通过伪代码的方式来讲解下这两种启动方式的区别，然后再具体分析下FreeRTOS的启动流程。

万事俱备，只欠东风
~~~~~~~~~~~~~~~~~~

第一种我称之为万事俱备，只欠东风法。这种方法是在main函数中将硬件初始化，RTOS系统初始化，所有任务的创建这些都弄好，这个我称之为万事都已经准备好。
最后只欠一道东风，即启动RTOS的调度器，开始多任务的调度，具体的伪代码实现见 代码清单15_1_ 。

.. code-block:: c
    :caption: 代码清单‑1万事俱备，只欠东风法伪代码实现
    :name: 代码清单15_1
    :linenos:

    int main (void)
    {
    /* 硬件初始化 */
        HardWare_Init();(1)

    /* RTOS 系统初始化 */
        RTOS_Init();(2)

    /* 创建任务1，但任务1不会执行，因为调度器还没有开启 */(3)
        RTOS_TaskCreate(Task1);
    /* 创建任务2，但任务2不会执行，因为调度器还没有开启 */
    RTOS_TaskCreate(Task2);

    /* ......继续创建各种任务 */

    /* 启动RTOS，开始调度 */
        RTOS_Start();(4)
    }

    voidTask1( void *arg )(5)
    {
    while (1)
        {
    /* 任务实体，必须有阻塞的情况出现 */
        }
    }

    voidTask1( void *arg )(6)
    {
    while (1)
        {
    /* 任务实体，必须有阻塞的情况出现 */
        }
    }

-   代码清单15_1_ **(1)**\ ：硬件初始化。硬件初始化这一步还属于裸机的范畴，我们可以把需要使用到的硬件都初始化好而且测试好，确保无误。

-   代码清单15_1_  **(2)**\ ：RTOS系统初始化。比如RTOS里面的全局变量的初始化，空闲任务的创建等。不同的RTOS，它们的初始化有细微的差别。

-   代码清单15_1_  **(3)**\ ：创建各种任务。这里把所有要用到的任务都创建好，但还不会进入调度，因为这个时候RTOS的调度器还没有开启。

-   代码清单15_1_  **(4)**\ ：启动RTOS调度器，开始任务调度。这个时候调度器就从刚刚创建好的任务中选择一个优先级最高的任务开始运行。

-   代码清单15_1_  **(5)(6)**\ ：任务实体通常是一个不带返回值的无限循环的C函数，函数体必须有阻塞的情况出现，不然任务（如果优先
    权恰好是最高）会一直在while循环里面执行，导致其他任务没有执行的机会。

小心翼翼，十分谨慎
~~~~~~~~~~~~~~~~~~

第二种我称之为小心翼翼，十分谨慎法。这种方法是在main函数中将硬件和RTOS系统先初始化好，然后创建一个启动任务后就启动调度器，
然后在启动任务里面创建各种应用任务，当所有任务都创建成功后，启动任务把自己删除，具体的伪代码实现见 代码清单15_2_。

.. code-block:: c
    :caption: 代码清单15_2小心翼翼，十分谨慎法伪代码实现
    :name: 代码清单15_2
    :linenos:

    int main (void)
    {
    /* 硬件初始化 */
        HardWare_Init();(1)
    
    /* RTOS 系统初始化 */
        RTOS_Init();(2)
    
    /* 创建一个任务 */
        RTOS_TaskCreate(AppTaskCreate);(3)
    
    /* 启动RTOS，开始调度 */
        RTOS_Start();(4)
    }
    
    /* 起始任务，在里面创建任务 */
    voidAppTaskCreate( void *arg )(5)
    {
    /* 创建任务1，然后执行 */
        RTOS_TaskCreate(Task1);(6)
    
    /* 当任务1阻塞时，继续创建任务2，然后执行 */
        RTOS_TaskCreate(Task2);
    
    /* ......继续创建各种任务 */
    
    /* 当任务创建完成，删除起始任务 */
        RTOS_TaskDelete(AppTaskCreate);(7)
    }
    
    void Task1( void *arg )(8)
    {
    while (1)
        {
    /* 任务实体，必须有阻塞的情况出现 */
        }
    }
    void Task2( void *arg )(9)
    {
    while (1)
        {
    /* 任务实体，必须有阻塞的情况出现 */
        }
    }



代码清单15_2_ **(1)**\ ：硬件初始化。来到硬件初始化这一步还属于裸机的范畴，我们可以把需要使用到的硬件都初始化好而且测试好，确保无误。

代码清单15_2_ **(2)**\ ：RTOS系统初始化。比如RTOS里面的全局变量的初始化，空闲任务的创建等。不同的RTOS，它们的初始化有细微的差别。

代码清单15_2_ **(3)**\ ：创建一个开始任务。然后在这个初始任务里面创建各种应用任务。

代码清单15_2_ **(4)**\ ：启动RTOS调度器，开始任务调度。这个时候调度器就去执行刚刚创建好的初始任务。

代码清单15_2_ **(5)**\ ：我们通常说任务是一个不带返回值的无限循环的C函数，但是因为初始任务的特殊性，它不能是无限循环的，只执行一次后就关闭。在初始任务里面我们创建我们需要的各种任务。

代码清单15_2_ **(6)**\ ：创建任务。每创建一个任务后它都将进入就绪态，系统会进行一次调度，如果新创建的任务的优先级比初始任务的优先级高的话，那将去执行新创建的任务，当新的任务阻塞时再回到初始任务被打断的地方继续执行。反之，则继续往下创建新的任务，直到所有任务创建完成。

代码清单15_2_ **(7)**\ ：各种应用任务创建完成后，初始任务自己关闭自己，使命完成。

代码清单15_2_ **(8)(9)**\ ：任务实体通常是一个不带返回值的无限循环的C函数，函数体必须有阻塞的情况出现，不然任务（如果优先权恰好是最高）会一直在while循环里面执行，其它任务没有执行的机会。

孰优孰劣
~~~~~~~~~~~~

那有关这两种方法孰优孰劣？我暂时没发现，我个人还是比较喜欢使用第一种。LiteOS和ucos第一种和第二种都可以使用，由用户选择，RT-Thread和FreeRTOS则默认使用第二种。接下来我们详细讲解下FreeRTOS的启动流程。


FreeRTOS的启动流程
~~~~~~~~~~~~~~~~~~~~~~~~~~~~


我们知道，在系统上电的时候第一个执行的是启动文件里面由汇编编写的复位函数Reset_Handler，具体见 代码清单15_3_ 。复位函数的最后会调用C库函数__main，具体见 代码清单15_3_ 。__main函数的主要工作是初始化系统的堆和栈，最后调用C中的main函数，从而去到C的世界。

.. code-block:: c
    :caption: 代码清单15_3 Reset_Handler函数
    :name: 代码清单15_3
    :linenos:

    Reset_Handler   PROC
    EXPORT  Reset_Handler             [WEAK]
    IMPORT  __main
    IMPORT  SystemInit
    LDRR0, =SystemInit
                    BLX     R0
    LDRR0, =__main
                    BX      R0
    ENDP


创建任务xTaskCreate()函数
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在main()函数中，我们直接可以对FreeRTOS进行创建任务操作，因为FreeRTOS会自动帮我们做初始化的事情，比如初始化堆内存。FreeRTOS的简单方便是在别的实时操作系统上都没有的，像RT-Tharead，需要做很多事情，具体可以看野火出版的另一本书《RT-
Thread内核实现与应用开发实战—基于I.MX RT系列芯片》；华为LiteOS也需要我们用户进行初始化内核，具体可以看野火出版的另一本书籍华为LiteOS《华为LiteOS内核实现与应用开发实战—基于I.MX RT系列芯片》。

这种简单的特点使得FreeRTOS在初学的时候变得很简单，我们自己在main()函数中直接初始化我们的板级外设——BSP_Init()，然后进行任务的创建即可——xTaskCreate()，在任务创建中，
FreeRTOS会帮我们进行一系列的系统初始化，在创建任务的时候，会帮我们初始化堆内存，具体见 代码清单15_4_ 。

.. code-block:: c
    :caption: 代码清单15_4 xTaskCreate函数内部进行堆内存初始化
    :name: 代码清单15_4
    :linenos:

    BaseType_t xTaskCreate(TaskFunction_t pxTaskCode,
    const char * const pcName,
    const uint16_t usStackDepth,
    void * const pvParameters,
                            UBaseType_t uxPriority,
                            TaskHandle_t * const pxCreatedTask )
    {
    if ( pxStack != NULL ) {
    /* 分配任务控制块内存 */
            pxNewTCB = ( TCB_t * ) pvPortMalloc( sizeof( TCB_t ) ); (1)


    if ( pxNewTCB != NULL ) {
    /*将堆栈位置存储在TCB中。*/
                pxNewTCB->pxStack = pxStack;
            }
        }
    /*
    省略代码
        ......
        */
    }

    /* 分配内存函数 */
    void *pvPortMalloc( size_t xWantedSize )
    {
        BlockLink_t *pxBlock, *pxPreviousBlock, *pxNewBlockLink;
    void *pvReturn = NULL;

        vTaskSuspendAll();
        {

    /*如果这是对malloc的第一次调用，那么堆将需要初始化来设置空闲块列表。*/
    if ( pxEnd == NULL ) {
                prvHeapInit();		(2)
            } else {
                mtCOVERAGE_TEST_MARKER();
            }
    /*
    省略代码
            ......
            */

        }
    }



从 代码清单15_4_ 的\ **(1)(2)**\ 中，我们知道：在未初始化内存的时候一旦调用了xTaskCreate()函数，FreeRTOS就会帮我们自动进行内存的初始化，内存的初始化具体见 代码清单15_5_。注意，此函数是FreeRTOS内部调用的，目前我们暂时不用管这个函数的实现，在后面我们会仔细
讲解FreeRTOS的内存管理相关知识，现在我们知道FreeRTOS会帮我们初始话系统要用的东西即可。

.. code-block:: c
    :caption: 代码清单15_5 prvHeapInit()函数定义
    :name: 代码清单15_5
    :linenos:

    static void prvHeapInit( void )
    {
        BlockLink_t *pxFirstFreeBlock;
    uint8_t *pucAlignedHeap;
    size_t uxAddress;
    size_t xTotalHeapSize = configTOTAL_HEAP_SIZE;		
    
    
        uxAddress = ( size_t ) ucHeap;
        /*确保堆在正确对齐的边界上启动。*/
    if ( ( uxAddress & portBYTE_ALIGNMENT_MASK ) != 0 ) {	
            uxAddress += ( portBYTE_ALIGNMENT - 1 );
            uxAddress &= ~( ( size_t ) portBYTE_ALIGNMENT_MASK );
            xTotalHeapSize -= uxAddress - ( size_t ) ucHeap;
        }
    
        pucAlignedHeap = ( uint8_t * ) uxAddress;			
    
    /* xStart用于保存指向空闲块列表中第一个项目的指针。
    void用于防止编译器警告*/
        xStart.pxNextFreeBlock = ( void * ) pucAlignedHeap;		
        xStart.xBlockSize = ( size_t ) 0;
    
    
        /* pxEnd用于标记空闲块列表的末尾，并插入堆空间的末尾。*/
        uxAddress = ( ( size_t ) pucAlignedHeap ) + xTotalHeapSize;
        uxAddress -= xHeapStructSize;
        uxAddress &= ~( ( size_t ) portBYTE_ALIGNMENT_MASK );
        pxEnd = ( void * ) uxAddress;
        pxEnd->xBlockSize = 0;
        pxEnd->pxNextFreeBlock = NULL;
    
    
        /*首先，有一个空闲块，其大小可以占用整个堆空间，减去pxEnd占用的空间。*/
        pxFirstFreeBlock = ( void * ) pucAlignedHeap;
        pxFirstFreeBlock->xBlockSize = uxAddress - ( size_t ) pxFirstFreeBlock;
        pxFirstFreeBlock->pxNextFreeBlock = pxEnd;
    
    /*只存在一个块 - 它覆盖整个可用堆空间。因为是刚初始化的堆内存*/
        xMinimumEverFreeBytesRemaining = pxFirstFreeBlock->xBlockSize;
        xFreeBytesRemaining = pxFirstFreeBlock->xBlockSize;
    
    
        xBlockAllocatedBit = ( ( size_t ) 1 ) << ( ( sizeof( size_t ) * 
                    heapBITS_PER_BYTE ) - 1 );


vTaskStartScheduler()函数
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在创建完任务的时候，我们需要开启调度器，因为创建仅仅是把任务添加到系统中，还没真正调度，并且空闲任务也没实现，定时器任务也没实现，这些都是在开启调度函数vTaskStartScheduler()中实现的。为什么要空闲任务？因为FreeRTOS一旦启动，就必须要保证系统中每时每刻都有一个任务处于运行态
（Runing），并且空闲任务不可以被挂起与删除，空闲任务的优先级是最低的，以便系统中其他任务能随时抢占空闲任务的CPU使用权。这些都是系统必要的东西，也无需用户自己实现，FreeRTOS全部帮我们搞定了。处理完这些必要的东西之后，系统才真正开始启动，具体见 代码清单15_6_ 高亮部分。

.. code-block:: c
    :caption: 代码清单15_6 vTaskStartScheduler()函数
    :emphasize-lines: 35-49
    :name: 代码清单15_6
    :linenos:

    /*-----------------------------------------------------------*/
 
    void vTaskStartScheduler( void )
    {
        BaseType_t xReturn;
    
    /*添加空闲任务*/
    #if( configSUPPORT_STATIC_ALLOCATION == 1 )
        {
            StaticTask_t *pxIdleTaskTCBBuffer = NULL;
            StackType_t *pxIdleTaskStackBuffer = NULL;
    uint32_t ulIdleTaskStackSize;
    
    /* 空闲任务是使用用户提供的RAM创建的 - 获取
    然后RAM的地址创建空闲任务。这是静态创建任务，我们不用管*/
            vApplicationGetIdleTaskMemory( &pxIdleTaskTCBBuffer,
    &pxIdleTaskStackBuffer,
    &ulIdleTaskStackSize );
            xIdleTaskHandle = xTaskCreateStatic(prvIdleTask,
    "IDLE",
            ulIdleTaskStackSize,
        ( void * ) NULL,
    ( tskIDLE_PRIORITY | portPRIVILEGE_BIT ),
                                                pxIdleTaskStackBuffer,
                                                pxIdleTaskTCBBuffer );
    
    if ( xIdleTaskHandle != NULL ) {
                xReturn = pdPASS;
            } else {
                xReturn = pdFAIL;
            }
        }
    #else	/* 这里才是动态创建idle任务 */
        {
    /* 使用动态分配的RAM创建空闲任务。 */
            xReturn = xTaskCreate(	prvIdleTask,			(1)	
    "IDLE", configMINIMAL_STACK_SIZE,
                                    ( void * ) NULL,
                                    ( tskIDLE_PRIORITY | portPRIVILEGE_BIT ),
    &xIdleTaskHandle );
        }
    #endif
    
    #if ( configUSE_TIMERS == 1 )
        {
    /* 如果使能了 configUSE_TIMERS宏定义
    表明使用定时器，需要创建定时器任务*/
    if ( xReturn == pdPASS ) {
                xReturn = xTimerCreateTimerTask();			(2)
            } else {
                mtCOVERAGE_TEST_MARKER();
            }
        }
    #endif/* configUSE_TIMERS */

    if ( xReturn == pdPASS ) {
    /* 此处关闭中断，以确保不会发生中断
    在调用xPortStartScheduler（）之前或期间。堆栈的
    创建的任务包含打开中断的状态
    因此，当第一个任务时，中断将自动重新启用
    开始运行。 */
            portDISABLE_INTERRUPTS();

    #if ( configUSE_NEWLIB_REENTRANT == 1 )
            {
    /* 不需要理会，这个宏定义没打开 */
                _impure_ptr = &( pxCurrentTCB->xNewLib_reent );
            }
    #endif/* configUSE_NEWLIB_REENTRANT */

            xNextTaskUnblockTime = portMAX_DELAY;		
            xSchedulerRunning = pdTRUE;			(3)
            xTickCount = ( TickType_t ) 0U;			

    /* 如果定义了configGENERATE_RUN_TIME_STATS，则以下内容
    必须定义宏以配置用于生成的计时器/计数器
    运行时计数器时基。目前没启用该宏定义 */
            portCONFIGURE_TIMER_FOR_RUN_TIME_STATS();

    /* 调用xPortStartScheduler函数配置相关硬件
    如滴答定时器、FPU、pendsv等		*/
    if ( xPortStartScheduler() != pdFALSE ) {	(4)
    /* 如果xPortStartScheduler函数启动成功，则不会运行到这里 */
            } else {
    /* 不会运行到这里，除非调用 xTaskEndScheduler() 函数 */
            }
        } else {
    /* 只有在内核无法启动时才会到达此行，
    因为没有足够的堆内存来创建空闲任务或计时器任务。
        此处使用了断言，会输出错误信息，方便错误定位 */
            configASSERT( xReturn != errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY );
        }

    /* 如果INCLUDE_xTaskGetIdleTaskHandle设置为0，则防止编译器警告，
        这意味着在其他任何地方都不使用xIdleTaskHandle。暂时不用理会 */
        ( void ) xIdleTaskHandle;
    }
    /*-----------------------------------------------------------*/


代码清单15_6_ **(1)**\ ：动态创建空闲任务（IDLE），因为现在我们不使用静态创建，这个configSUPPORT_STATIC_ALLOCATION宏定义为0，只能是动态创建空闲任务，并且空闲任务的优先级与堆栈大小都在FreeRTOSConfig.h中由用户定义，空闲任务的任务句柄存
放在静态变量xIdleTaskHandle中，用户可以调用API函数xTaskGetIdleTaskHandle()获得空闲任务句柄。

代码清单15_6_ **(2)**\ ：如果在FreeRTOSConfig.h中使能了configUSE_TIMERS这个宏定义，那么需要创建一个定时器任务，这个定时器任务也是调用xTaskCreate()函数完成创建，过程十分简单，这也是系统的初始化内容，在调度器启动的过程中发现必要初始化的东西，
FreeRTOS就会帮我们完成，真的对开发者太友好了，xTimerCreateTimerTask()函数具体见 代码清单15_7_ 高亮部分。


.. code-block:: c
    :caption: 代码清单15_7 xTimerCreateTimerTask源码
    :emphasize-lines: 37-43
    :name: 代码清单15_7
    :linenos:

    BaseType_t xTimerCreateTimerTask( void )
    {
        BaseType_t xReturn = pdFAIL;
    
    /* 检查使用了哪些活动计时器的列表，以及
    用于与计时器服务通信的队列，已经
    初始化。*/
        prvCheckForValidListAndQueue();
    
    if ( xTimerQueue != NULL ) {
    #if( configSUPPORT_STATIC_ALLOCATION == 1 )
            {
    /* 这是静态创建的，无需理会 */
                StaticTask_t *pxTimerTaskTCBBuffer = NULL;
                StackType_t *pxTimerTaskStackBuffer = NULL;
    uint32_t ulTimerTaskStackSize;
    
    vApplicationGetTimerTaskMemory(&pxTimerTaskTCBBuffer,
                    &pxTimerTaskStackBuffer,
                    &ulTimerTaskStackSize );
                xTimerTaskHandle = xTaskCreateStatic(prvTimerTask,
                    "Tmr Svc",
                    ulTimerTaskStackSize,
                    NULL,
        ( ( UBaseType_t ) configTIMER_TASK_PRIORITY ) |
                    portPRIVILEGE_BIT,
                    pxTimerTaskStackBuffer,
                    pxTimerTaskTCBBuffer );
    
    if ( xTimerTaskHandle != NULL )
                {
                    xReturn = pdPASS;
                }
            }
    #else
            {/* 这是才是动态创建定时器任务*/
                xReturn = xTaskCreate(prvTimerTask,
    "Tmr Svc",
                configTIMER_TASK_STACK_DEPTH,
                NULL,
                ( ( UBaseType_t ) configTIMER_TASK_PRIORITY ) |
                portPRIVILEGE_BIT,
    &xTimerTaskHandle );
            }
    #endif/* configSUPPORT_STATIC_ALLOCATION */
        } else {
            mtCOVERAGE_TEST_MARKER();
        }
    
        configASSERT( xReturn );


代码清单15_6_ **(3)**\ ：xSchedulerRunning等于pdTRUE，表示调度器开始运行了，而xTickCount初始化需要初始化为0，这个xTickCount变量用于记录系统的时间，在节拍定时器（SysTick）中断服务函数中进行自加。

代码清单15_6_ **(4)**\ ：调用函数xPortStartScheduler()来启动系统节拍定时器（一般都是使用SysTick）并启动第一个任务。因为设置系统节拍定时器涉及到硬件特性，因此函数xPortStartScheduler()由移植层提供（在port.c文件实现），不同的硬件架构
，这个函数的代码也不相同，在ARM_CM3中，使用SysTick作为系统节拍定时器。有兴趣可以看看xPortStartScheduler()的源码内容，下面我只是简单介绍一下相关知识。

在Cortex-M3架构中，FreeRTOS为了任务启动和任务切换使用了三个异常：SVC、PendSV和SysTick：

SVC（系统服务调用，亦简称系统调用）用于任务启动，有些操作系统不允许应用程序直接访问硬件，而是通过提供一些系统服务函数，用户程序使用 SVC 发出对系统服务函数的呼叫请求，以这种方法调用它们来间接访问硬件，它就会产生一个 SVC 异常。

PendSV（可挂起系统调用）用于完成任务切换，它是可以像普通的中断一样被挂起的，它的最大特性是如果当前有优先级比它高的中断在运行，PendSV会延迟执行，直到高优先级中断执行完毕，这样子产生的PendSV中断就不会打断其他中断的运行。

SysTick用于产生系统节拍时钟，提供一个时间片，如果多个任务共享同一个优先级，则每次SysTick中断，下一个任务将获得一个时间片。关于详细的SVC、PendSV异常描述，推荐《Cortex-M3权威指南》一书的“异常”部分。

这里将PendSV和SysTick异常优先级设置为最低，这样任务切换不会打断某个中断服务程序，中断服务程序也不会被延迟，这样简化了设计，有利于系统稳定。有人可能会问，那SysTick的优先级配置为最低，那延迟的话系统时间会不会有偏差？答案是不会的，因为SysTick只是当次响应中断被延迟了，而Sys
Tick是硬件定时器，它一直在计时，这一次的溢出产生中断与下一次的溢出产生中断的时间间隔是一样的，至于系统是否响应还是延迟响应，这个与SysTick无关，它照样在计时。

main函数
^^^^^^

当我们拿到一个移植好FreeRTOS的例程的时候，不出意外，你首先看到的是main函数，当你认真一看main函数里面只是创建并启动一些任务和硬件初始化，具体见 代码清单15_8_ 。而系统初始化这些工作不需要我们实现，因为FreeRTOS在我们使用创建与开启调度的时候就已经偷偷帮我们做完了，如果只是使用F
reeRTOS的话，无需关注FreeRTOS API函数里面的实现过程，但是我们还是建议需要深入了解FreeRTOS然后再去使用，避免出现问题。

.. code-block:: c
    :caption: 代码清单15_8 main函数
    :name: 代码清单15_8
    :linenos:

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
        BSP_Init();					(1)
        printf("这是一个[野火]-全系列开发板-FreeRTOS-多任务创建实验!\r\n");
    /* 创建AppTaskCreate任务 */				(2)
        xReturn = xTaskCreate((TaskFunction_t )AppTaskCreate,/* 任务入口函数 */
                            (const char*    )"AppTaskCreate",/* 任务名字 */
                            (uint16_t       )512,  /* 任务栈大小 */
                            (void*          )NULL,/* 任务入口函数参数 */
                            (UBaseType_t    )1, /* 任务的优先级 */
    (TaskHandle_t*)&AppTaskCreate_Handle);/*任务控制块指针*/
    /* 启动任务调度 */
    if (pdPASS == xReturn)
    vTaskStartScheduler();   /* 启动任务，开启调度 */	(3)
    else
    return -1;					(4)

    while (1);  /* 正常不会执行到这里 */
    }



代码清单15_8_ **(1)**\ ：开发板硬件初始化，FreeRTOS系统初始化是经在创建任务与开启调度器的时候完成的。

代码清单15_8_ **(2)**\ ：在AppTaskCreate中创建各种应用任务，具体见 代码清单15_9_ 。



.. code-block:: c
    :caption: 代码清单15_9 AppTaskCreate函数
    :name: 代码清单15_9
    :linenos:

    /***********************************************************************
    * @ 函数名： AppTaskCreate
    * @ 功能说明：为了方便管理，所有的任务创建函数都放在这个函数里面
    * @ 参数：无
    * @ 返回值：无
    **********************************************************************/
    static void AppTaskCreate(void)
    {
        BaseType_t xReturn = pdPASS;/* 定义一个创建信息返回值，默认为pdPASS */
    
        taskENTER_CRITICAL();           //进入临界区
    
    /* 创建LED_Task任务 */
        xReturn = xTaskCreate((TaskFunction_t )LED1_Task, /* 任务入口函数 */
                            (const char*    )"LED1_Task",/* 任务名字 */
                            (uint16_t       )512,   /* 任务栈大小 */
                            (void*          )NULL, /* 任务入口函数参数 */
                            (UBaseType_t    )2,	/* 任务的优先级 */
                            (TaskHandle_t*  )&LED1_Task_Handle);/* 任务控制块指针 */
    if (pdPASS == xReturn)
            printf("创建LED1_Task任务成功!\r\n");
    
    /* 创建LED_Task任务 */
        xReturn = xTaskCreate((TaskFunction_t )LED2_Task, /* 任务入口函数 */
                            (const char*    )"LED2_Task",/* 任务名字 */
                            (uint16_t       )512,   /* 任务栈大小 */
                            (void*          )NULL, /* 任务入口函数参数 */
                            (UBaseType_t    )3,	/* 任务的优先级 */
                            (TaskHandle_t*  )&LED2_Task_Handle);/* 任务控制块指针 */
    if (pdPASS == xReturn)
            printf("创建LED2_Task任务成功!\r\n");
    
        vTaskDelete(AppTaskCreate_Handle); //删除AppTaskCreate任务

 
        taskEXIT_CRITICAL();            //退出临界区
    }

当创建的应用任务的优先级比AppTaskCreate任务的优先级高、低或者相等时候，程序是如何执行的？假如像我们代码一样在临界区创建任务，任务只能在退出临界区的时候才执行最高优先级任务。假如没使用临界区的话，就会分三种情况：1、应用任务的优先级比初始任务的优先级高，那创建完后立马去执行刚刚创建的应用
任务，当应用任务被阻塞时，继续回到初始任务被打断的地方继续往下执行，直到所有应用任务创建完成，最后初始任务把自己删除，完成自己的使命；2、应用任务的优先级与初始任务的优先级一样，那创建完后根据任务的时间片来执行，直到所有应用任务创建完成，最后初始任务把自己删除，完成自己的使命；3、应用任务的优先级比
初始任务的优先级低，那创建完后任务不会被执行，如果还有应用任务紧接着创建应用任务，如果应用任务的优先级出现了比初始任务高或者相等的情况，请参考1和2的处理方式，直到所有应用任务创建完成，最后初始任务把自己删除，完成自己的使命。

代码清单15_8_ **(3)(4)**\ ：在启动任务调度器的时候，假如启动成功的话，任务就不会有返回了，假如启动没成功，则通过LR寄存器指定的地址退出，在创建AppTaskCreate任务的时候，任务栈对应LR寄存器指向是任务退出函数prvTaskExitError()，该函数里面是一个死循环，这代表着假如创建任务
没成功的话，就会进入死循环，该任务也不会运行。
