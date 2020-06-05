.. vim: syntax=rst

任务延时列表的实现
==================

在本章之前，为了实现任务的阻塞延时，在任务控制块中内置了一个延时变量xTicksToDelay。每当任务需要延时的时候，
就初始化xTicksToDelay需要延时的时间，然后将任务挂起，这里的挂起只是将任务在优先级位图表uxTopReadyPriority
中对应的位清零，并不会将任务从就绪列表中删除。当每次时基中断（SysTick中断）来临时，就扫描就绪列表中的每个任
务的xTicksToDelay，如果xTicksToDelay大于0则递减一次，然后判断xTicksToDelay是否为0，如果为0则表示延时时间
到，将该任务就绪（即将任务在优先级位图表uxTopReadyPriority中对应的位置位），然后进行任务切换。这种延时的缺
点是，在每个时基中断中需要对所有任务都扫描一遍，费时，优点是容易理解。之所以先这样讲解是为了慢慢地过度到
FreeRTOS任务延时列表的讲解。

任务延时列表的工作原理
~~~~~~~~~~~~~~~~~~~~~~

在FreeRTOS中，有一个任务延时列表（实际上有两个，为了方便讲解原理，我们假装合并为一个，其实两个的作用是一样的），
当任务需要延时的时候，则先将任务挂起，即先将任务从就绪列表删除，然后插入到任务延时列表，同时更新下一个任务的
解锁时刻变量：xNextTaskUnblockTime的值。

xNextTaskUnblockTime的值等于系统时基计数器的值xTickCount加上任务需要延时的值xTicksToDelay。当系统时基计数
器xTickCount的值与xNextTaskUnblockTime相等时，就表示有任务延时到期了，需要将该任务就绪。与RT-Thread和μC/OS
在解锁延时任务时要扫描定时器列表这种时间不确定性的方法相比，FreeRTOS这个xNextTaskUnblockTime全局变量设计的非常巧妙。

任务延时列表表维护着一条双向链表，每个节点代表了正在延时的任务，节点按照延时时间大小做升序排列。当每次时基中
断（SysTick中断）来临时，就拿系统时基计数器的值xTickCount与下一个任务的解锁时刻变量xNextTaskUnblockTime的
值相比较，如果相等，则表示有任务延时到期，需要将该任务就绪，否则只是单纯地更新系统时基计数器xTickCount的值，
然后进行任务切换。

实现任务延时列表
~~~~~~~~~~~~~~~~

接下来具体讲解下FreeRTOS中任务延时列表的实现。

定义任务延时列表
^^^^^^^^^^^^^^^^

任务延时列表在task.c中定义，具体见 代码清单:延时-1_。

.. code-block:: c
    :caption: 代码清单:延时-1任务延时列表定义
    :name: 代码清单:延时-1
    :linenos:

    static List_t xDelayedTaskList1;(1)
    static List_t xDelayedTaskList2;(2)
    static List_t * volatile pxDelayedTaskList;(3)
    static List_t * volatile pxOverflowDelayedTaskList;(4)


-   代码清单:延时-1_ **(1)(2)**\ ：FreeRTOS定义了两个任务延时列表，当系统时基计数器xTickCount没有溢出时，用一条
    列表，当xTickCount溢出后，用另外一条列表。

-   代码清单:延时-1_ **(3)**\ ：任务延时列表指针，指向xTickCount没有溢出时使用的那条列表。

-   代码清单:延时-1_ **(7)**\ ：任务延时列表指针，指向xTickCount溢出时使用的那条列表。

任务延时列表初始化
^^^^^^^^^^^^^^^^^^

任务延时列表属于任务列表的一种，在prvInitialiseTaskLists()函数中初始化，具体见 代码清单:延时-2_ 的高亮部分。

.. code-block:: c
    :caption: 代码清单:延时-2prvInitialiseTaskLists()函数
    :emphasize-lines: 14-18
    :name: 代码清单:延时-2
    :linenos:

    /* 初始化任务相关的列表 */
    void prvInitialiseTaskLists( void )
    {
        UBaseType_t uxPriority;

    /* 初始化就绪列表 */
    for ( uxPriority = ( UBaseType_t ) 0U;
                uxPriority < ( UBaseType_t ) configMAX_PRIORITIES;
                uxPriority++ )
        {
            vListInitialise( &( pxReadyTasksLists[ uxPriority ] ) );
        }

        vListInitialise( &xDelayedTaskList1 );
        vListInitialise( &xDelayedTaskList2 );

        pxDelayedTaskList = &xDelayedTaskList1;
        pxOverflowDelayedTaskList = &xDelayedTaskList2;
    }


定义xNextTaskUnblockTime
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

xNextTaskUnblockTime是一个在task.c中定义的静态变量，用于表示下一个任务的解锁时刻。xNextTaskUnblockTime的值等
于系统时基计数器的值xTickCount加上任务需要延时值xTicksToDelay。当系统时基计数器xTickCount的值与
xNextTaskUnblockTime相等时，就表示有任务延时到期了，需要将该任务就绪。

初始化xNextTaskUnblockTime
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

xNextTaskUnblockTime在vTaskStartScheduler()函数中初始化为portMAX_DELAY（portMAX_DELAY是一个portmacro.h中
定义的宏，默认为0xffffffffUL），具体实现见 代码清单:延时-3_ 的高亮部分。

.. code-block:: c
    :caption: 代码清单:延时-3初始化xNextTaskUnblockTime
    :emphasize-lines: 23
    :name: 代码清单:延时-3
    :linenos:

    void vTaskStartScheduler( void )
    {
    /*==================创建空闲任务start=========================*/
        TCB_t *pxIdleTaskTCBBuffer = NULL;
        StackType_t *pxIdleTaskStackBuffer = NULL;
    uint32_t ulIdleTaskStackSize;

    /* 获取空闲任务的内存：任务栈和任务TCB */
        vApplicationGetIdleTaskMemory( &pxIdleTaskTCBBuffer,
    &pxIdleTaskStackBuffer,
    &ulIdleTaskStackSize );

        xIdleTaskHandle =
    xTaskCreateStatic( (TaskFunction_t)prvIdleTask,
    (char *)"IDLE",
    (uint32_t)ulIdleTaskStackSize ,
    (void *) NULL,
    (UBaseType_t) tskIDLE_PRIORITY,
    (StackType_t *)pxIdleTaskStackBuffer,
    (TCB_t *)pxIdleTaskTCBBuffer );
    /*======================创建空闲任务end===================*/

        xNextTaskUnblockTime = portMAX_DELAY;

        xTickCount = ( TickType_t ) 0U;

    /* 启动调度器 */
    if ( xPortStartScheduler() != pdFALSE )
        {
    /* 调度器启动成功，则不会返回，即不会来到这里 */
        }
    }


修改代码，支持任务延时列表
~~~~~~~~~~~~~~~~~~~~~~~~~~

接下来我们在上一章的代码上，继续迭代修改，从而支持任务延时列表。

修改vTaskDelay()函数
^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: c
    :caption: 代码清单:延时-4vTaskDelay()函数
    :emphasize-lines: 8-12
    :name: 代码清单:延时-4
    :linenos:

    void vTaskDelay( const TickType_t xTicksToDelay )
    {
        TCB_t *pxTCB = NULL;

    /* 获取当前任务的TCB */
        pxTCB = pxCurrentTCB;

    /* 设置延时时间 */
    //pxTCB->xTicksToDelay = xTicksToDelay;(1)

    /* 将任务插入到延时列表 */
        prvAddCurrentTaskToDelayedList( xTicksToDelay );(2)

    /* 任务切换 */
        taskYIELD();
    }


-   代码清单:延时-4_ **(1)**\ ：从本章开始，添加了任务的延时列表，延时的时候不用再依赖任务TCB中内置的延时变量xTicksToDelay。

-   代码清单:延时-4_ **(2)**\ ：将任务插入到延时列表。函数prvAddCurrentTaskToDelayedList()在task.c中定义，具
    体实现见 代码清单:延时-5_。

prvAddCurrentTaskToDelayedList()函数
''''''''''''''''''''''''''''''''''''''''

.. code-block:: c
    :caption: 代码清单:延时-5prvAddCurrentTaskToDelayedList()函数
    :name: 代码清单:延时-5
    :linenos:

    static void prvAddCurrentTaskToDelayedList( TickType_t xTicksToWait )
    {
        TickType_t xTimeToWake;

    /* 获取系统时基计数器xTickCount的值 */
    const TickType_t xConstTickCount = xTickCount;(1)

    /* 将任务从就绪列表中移除 */(2)
    if ( uxListRemove( &( pxCurrentTCB->xStateListItem ) )
                == ( UBaseType_t ) 0 )
        {
    /* 将任务在优先级位图中对应的位清除 */
            portRESET_READY_PRIORITY( pxCurrentTCB->uxPriority,
                                    uxTopReadyPriority );
        }

    /* 计算任务延时到期时，系统时基计数器xTickCount的值是多少 */(3)
        xTimeToWake = xConstTickCount + xTicksToWait;

    /* 将延时到期的值设置为节点的排序值 */(4)
        listSET_LIST_ITEM_VALUE( &( pxCurrentTCB->xStateListItem ),
                                xTimeToWake );

    /* 溢出 */(5)
    if ( xTimeToWake < xConstTickCount )
        {
            vListInsert( pxOverflowDelayedTaskList,
    &( pxCurrentTCB->xStateListItem ) );
        }
    else/* 没有溢出 */
        {

            vListInsert( pxDelayedTaskList,
    &( pxCurrentTCB->xStateListItem ) );(6)

    /* 更新下一个任务解锁时刻变量xNextTaskUnblockTime的值*/(7)
    if ( xTimeToWake < xNextTaskUnblockTime )
            {
                xNextTaskUnblockTime = xTimeToWake;
            }
        }
    }


-   代码清单:延时-5_ **(1)**\ ：获取系统时基计数器xTickCount的值，xTickCount是一个在task.c中定义的全局变量，
    用于记录SysTick的中断次数。

-   代码清单:延时-5_ **(2)**\ ：调用函数uxListRemove()将任务从就绪列表移除，uxListRemove()会返回当前链表下节
    点的个数，如果为0，则表示当前链表下没有任务就绪，则调用函数portRESET_READY_PRIORITY()将任务在优先级位图表
    uxTopReadyPriority中对应的位清除。因为FreeRTOS支持同一个优先级下可以有多个任务，所以在清除优先级位图表
    uxTopReadyPriority中对应的位时要判断下该优先级下的就绪列表是否还有其他的任务。目前为止，我们还没有支持同一
    个优先级下有多个任务的功能，这个功能我们将在下一章“支持时间片”里面实现。

-   代码清单:延时-5_ **(3)**\ ：计算任务延时到期时，系统时基计数器xTickCount的值是多少。

-   代码清单:延时-5_ **(4)**\ ：将任务延时到期的值设置为节点的排序值。将任务插入到延时列表时就是根据这个值来做
    升序排列的，最先延时到期的任务排在最前面。

-   代码清单:延时-5_ **(5)**\ ：xTimeToWake溢出，将任务插入到溢出任务延时列表。溢出？什么意思？xTimeToWake等于
    系统时基计数器xTickCount的值加上任务需要延时的时间xTicksToWait。举例：如果当前xTickCount的值等于0xfffffffdUL，
    xTicksToWait等于0x03，那么xTimeToWake = 0xfffffffdUL + 0x03 = 1，显然得出的值比任务需要延时的时间0x03还
    小，这肯定不正常，说明溢出了，这个时候需要将任务插入到溢出任务延时列表。

-   代码清单:延时-5_ **(6)**\ ：xTimeToWake没有溢出，则将任务插入到正常任务延时列表。

-   代码清单:延时-5_ **(7)**\：更新下一个任务解锁时刻变量xNextTaskUnblockTime的值。这一步很重要，在
    xTaskIncrementTick()函数中，我们只需要让系统时基计数器xTickCount与xNextTaskUnblockTime的值先比较
    就知道延时最快结束的任务是否到期。

修改xTaskIncrementTick()函数
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

xTaskIncrementTick()函数改动较大，具体见 代码清单:延时-6_ 的高亮部分。

.. code-block:: c
    :caption: 代码清单:延时-6xTaskIncrementTick()函数
    :emphasize-lines: 9-45
    :name: 代码清单:延时-6
    :linenos:

    void xTaskIncrementTick( void )
    {
        TCB_t * pxTCB;
        TickType_t xItemValue;

        const TickType_t xConstTickCount = xTickCount + 1;(1)
        xTickCount = xConstTickCount;

        /* 如果xConstTickCount溢出，则切换延时列表 */(2)
        if( xConstTickCount == ( TickType_t ) 0U )
        {
            taskSWITCH_DELAYED_LISTS();
        }

        /* 最近的延时任务延时到期 */(3)
        if( xConstTickCount >= xNextTaskUnblockTime )
        {
            for( ;; )
            {
                if( listLIST_IS_EMPTY( pxDelayedTaskList ) != pdFALSE )(4)
                {
                    /* 延时列表为空，设置xNextTaskUnblockTime为可能的最大值 */
                    xNextTaskUnblockTime = portMAX_DELAY;
                    break;
                }
                else /* 延时列表不为空 */(5)
                {
                    pxTCB = ( TCB_t * ) listGET_OWNER_OF_HEAD_ENTRY( pxDelayedTaskList );
                    xItemValue = listGET_LIST_ITEM_VALUE( &( pxTCB->xStateListItem ) );(6)

                    /* 直到将延时列表中所有延时到期的任务移除才跳出for循环 */(7)
                    if( xConstTickCount < xItemValue )
                    {
                        xNextTaskUnblockTime = xItemValue;
                        break;
                    }

                    /* 将任务从延时列表移除，消除等待状态 */(8)
                    ( void ) uxListRemove( &( pxTCB->xStateListItem ) );

                    /* 将解除等待的任务添加到就绪列表 */
                    prvAddTaskToReadyList( pxTCB );(9)
                }
            }
        }/* xConstTickCount >= xNextTaskUnblockTime */

        /* 任务切换 */
        portYIELD();(10)
    }


-   代码清单:延时-6_ **(1)**\ ：更新系统时基计数器xTickCount的值。

-   代码清单:延时-6_ **(2)**\ ：如果系统时基计数器xTickCount溢出，则切换延时列表。taskSWITCH_DELAYED_LISTS()
    函数在task.c中定义，具体实现见 代码清单:延时-7_。

taskSWITCH_DELAYED_LISTS()函数
''''''''''''''''''''''''''''''''

.. code-block:: c
    :caption: 代码清单:延时-7taskSWITCH_DELAYED_LISTS()函数
    :name: 代码清单:延时-7
    :linenos:

    #define taskSWITCH_DELAYED_LISTS()
    {
        List_t *pxTemp;(1)
        pxTemp = pxDelayedTaskList;
        pxDelayedTaskList = pxOverflowDelayedTaskList;
        pxOverflowDelayedTaskList = pxTemp;
        xNumOfOverflows++;
        prvResetNextTaskUnblockTime();(2)
    }


-   代码清单:延时-7_ **(1)**\ ：切换延时列表，实际就是更换pxDelayedTaskList和pxOverflowDelayedTaskList这
    两个指针的指向。

-   代码清单:延时-7_ **(2)**\ ：复位xNextTaskUnblockTime的值。prvResetNextTaskUnblockTime()函数在task.c
    中定义，具体实现见 代码清单:延时-8_。

prvResetNextTaskUnblockTime函数

.. code-block:: c
    :caption: 代码清单:延时-8prvResetNextTaskUnblockTime函数
    :name: 代码清单:延时-8
    :linenos:

    static void prvResetNextTaskUnblockTime( void )
    {
        TCB_t *pxTCB;

        if ( listLIST_IS_EMPTY( pxDelayedTaskList ) != pdFALSE )
        {
            /* 当前延时列表为空，则设置xNextTaskUnblockTime等于最大值 */
            xNextTaskUnblockTime = portMAX_DELAY;
        }
        else
        {
            /* 当前列表不为空，则有任务在延时，则获取当前列表下第一个节点的排序值
            然后将该节点的排序值更新到xNextTaskUnblockTime */
            ( pxTCB ) = ( TCB_t * ) listGET_OWNER_OF_HEAD_ENTRY( pxDelayedTaskList );
            xNextTaskUnblockTime = listGET_LIST_ITEM_VALUE( &( ( pxTCB )->xStateListItem ) );
        }
    }


-   代码清单:延时-8_ **(1)**\ ：当前延时列表为空，则设置xNextTaskUnblockTime等于最大值。

-   代码清单:延时-8_ **(2)**\ ：当前列表不为空，则有任务在延时，则获取当前列表下第一个节点的排序值，然后将该
    节点的排序值更新到xNextTaskUnblockTime。

-   代码清单:延时-6_ **(3)**\ ：有任务延时到期，则进入下面的for循环，一一将这些延时到期的任务从延时列表移除。

-   代码清单:延时-6_ **(4)**\ ：延时列表为空，则将xNextTaskUnblockTime设置为最大值，然后跳出for循环。

-   代码清单:延时-6_ **(5)**\ ：延时列表不为空，则需要将延时列表里面延时到期的任务删除，并将它们添加到就绪列表。

-   代码清单:延时-6_ **(6)**\ ：取出延时列表第一个节点的排序辅助值。

-   代码清单:延时-6_ **(7)**\ ：直到将延时列表中所有延时到期的任务移除才跳出for循环。延时列表中有可能存在多个延时相等的任务。

-   代码清单:延时-6_ **(8)**\ ：将任务从延时列表移除，消除等待状态。

-   代码清单:延时-6_ **(9)**\ ：将解除等待的任务添加到就绪列表。

-   代码清单:延时-6_ **(10)**\ ：执行一次任务切换。

修改taskRESET_READY_PRIORITY()函数
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在没有添加任务延时列表之前，与任务相关的列表只有一个，就是就绪列表，无论任务在延时还是就绪都只能通过扫描就
绪列表来找到任务的TCB，从而实现系统调度。所以在上一章“支持多优先级”中，实现taskRESET_READY_PRIORITY()函
数的时候，不用先判断当前优先级下就绪列表中的链表的节点是否为0，而是直接把任务在优先级位图表uxTopReadyPriority
中对应的位清零。因为当前优先级下就绪列表中的链表的节点不可能为0，目前我们还没有添加其他列表来存放任务的TCB，只有一个就绪列表。

但是从本章开始，我们额外添加了延时列表，当任务要延时的时候，将任务从就绪列表移除，然后添加到延时列表，同时
将任务在优先级位图表uxTopReadyPriority中对应的位清除。在清除任务在优先级位图表uxTopReadyPriority中对应
的位的时候，与上一章不同的是需要判断就绪列表pxReadyTasksLists[]在当前优先级下对应的链表的节点是否为0，只
有当该链表下没有任务时才真正地将任务在优先级位图表uxTopReadyPriority中对应的位清零。

taskRESET_READY_PRIORITY()函数的具体修改见 代码清单:延时-9_ 的高亮部分。那什么情况下就绪列表的链表里面会有多
个任务节点？即同一优先级下有多个任务？这个就是我们下一章“支持时间片”要讲的内容。

.. code-block:: c
    :caption: 代码清单:延时-9taskRESET_READY_PRIORITY()函数
    :emphasize-lines: 4
    :name: 代码清单:延时-9
    :linenos:

    #if 1/*本章的实现方法*/
    #define taskRESET_READY_PRIORITY( uxPriority )
    {
        if( listCURRENT_LIST_LENGTH( &( pxReadyTasksLists[ ( uxPriority ) ] ) ) == ( UBaseType_t ) 0 )
        {
            portRESET_READY_PRIORITY( ( uxPriority ), ( uxTopReadyPriority ) );
        }
    }
    #else/* 上一章的实现方法*/
    #define taskRESET_READY_PRIORITY( uxPriority )
    {
        portRESET_READY_PRIORITY( ( uxPriority ), ( uxTopReadyPriority ) );
    }
    #endif


main函数
~~~~~~~~~

main函数与上一章一样，无需改动。

实验现象
~~~~~~~~

实验现象与上一章一样，虽说一样，但是实现延时的方法本质却变了，需要好好理解代码的实现，特别是当系统时基计数
器xTickCount发生溢出时，延时列表的更换是难点，这个可把我搞的云里雾里。
