.. vim: syntax=rst

临界段的保护
=============

什么是临界段
~~~~~~~~~~~~~~~~~~

临界段用一句话概括就是一段在执行的时候不能被中断的代码段。在FreeRTOS里面，这个临界段最常出现的就是对全局变量的操作，全局变量就好像是一个枪把子，谁都可以对他开枪，但是我开枪的时候，你就不能开枪，否则就不知道是谁命中了靶子。可能有人会说我可以在子弹上面做个标记，我说你能不能不要瞎扯淡。

那么什么情况下临界段会被打断？一个是系统调度，还有一个就是外部中断。在FreeRTOS，系统调度，最终也是产生PendSV中断，在PendSV Handler里面实现任务的切换，所以还是可以归结为中断。既然这样，FreeRTOS对临界段的保护最终还是回到对中断的开和关的控制。

Cortex-M内核快速关中断指令
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

为了快速地开关中断， Cortex-M内核专门设置了一条 CPS 指令，有 4 种用法，具体见 代码清单:临界段-1_。

.. code-block:: c
    :caption: 代码清单:临界段-1 CPS 指令用法
    :name: 代码清单:临界段-1
    :linenos:

    CPSID I ;PRIMASK=1     ;关中断
    CPSIE I ;PRIMASK=0     ;开中断
    CPSID F ;FAULTMASK=1   ;关异常
    CPSIE F ;FAULTMASK=0   ;开异常


-   代码清单:临界段-1_ 中PRIMASK和FAULTMAST是Cortex-M内核里面三个中断屏蔽寄存器中的两个，还有一个是BASEPRI，有关这三个寄
    存器的详细用法见 表:内核中断屏蔽寄存器组描述_。


.. list-table::
   :widths: 50 50
   :name: 表:内核中断屏蔽寄存器组描述
   :header-rows: 1


   * - 名字
     - 功能描述

   * - PRIMASK
     - 这是个只有单一比特的寄存器。在它被置1后，就关掉所有可屏蔽的异常，只剩下NMI和硬FAULT可以响应。它的缺省值是0，表示没有关中断。

   * - FAULTMASK
     - 这是个只有1 个位的寄存器。当它置1 时，只有NMI才能响应，所有其他的异常，甚至是硬FAULT，也通通闭嘴。它的缺省值也是0，表示没有关异常。

   * - BASEPRI
     - 这个寄存器最多有9位（由表达优先级的位数决定）。它定义了被屏蔽优先级的阈值。当它被设成
       某个值后，所有优先级号大于等于此值的中断都被关（优先级号越大，优先级越低）。但若被设成0，则不关闭任何中断，0也是缺省值。



但是，在FreeRTOS中，对中断的开和关是通过操作BASEPRI寄存器来实现的，即大于等于BASEPRI的值的中断会被屏蔽，小于BASEPRI的值的中断则不会被屏蔽，不受FreeRTOS管理。用户可以设置BASEPRI的值来选择性的给一些非常紧急的中断留一条后路。

关中断
~~~~~~~~~

FreeRTOS关中断的函数在portmacro.h中定义，分不带返回值和带返回值两种，具体实现见 代码清单:临界段-2_。

.. code-block:: guess
    :caption: 代码清单:临界段-2关中断函数
    :name: 代码清单:临界段-2
    :linenos:


    /* 不带返回值的关中断函数，不能嵌套，不能在中断里面使用 */(1)
    #define portDISABLE_INTERRUPTS() vPortRaiseBASEPRI()

    void vPortRaiseBASEPRI( void )
    {
    uint32_t ulNewBASEPRI = configMAX_SYSCALL_INTERRUPT_PRIORITY;(1)-1
        __asm
        {
            msr basepri, ulNewBASEPRI(1)-2
            dsb
            isb
        }
    }

    /* 带返回值的关中断函数，可以嵌套，可以在中断里面使用 */(2)
    #define portSET_INTERRUPT_MASK_FROM_ISR() ulPortRaiseBASEPRI()
    ulPortRaiseBASEPRI( void )
    {
        uint32_t ulReturn, ulNewBASEPRI = configMAX_SYSCALL_INTERRUPT_PRIORITY;(2)-1
        __asm
        {
            mrs ulReturn, basepri(2)-2
            msr basepri, ulNewBASEPRI(2)-3
            dsb
            isb
        }
        return ulReturn;(2)-4
    }

不带返回值的关中断函数
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

-   代码清单:临界段-2_ **(1)**\ ：不带返回值的关中断函数，不能嵌套，不能在中断里面使用。不带返回值的意思是：在往BASEPRI写
    入新的值的时候，不用先将BASEPRI的值保存起来，即不用管当前的中断状态是怎么样的，既然不用管当前的中断状态，也就意味着这样
    的函数不能在中断里面调用。

-   代码清单:临界段-2_ **(1)-1**\ ：configMAX_SYSCALL_INTERRUPT_PRIORITY是一个在FreeRTOSConfig.h中定义的宏，即要写入
    到BASEPRI寄存器的值。该宏默认定义为191，高四位有效，即等于0xb0，或者是11，即优先级大于等于11的中断都会被屏蔽，11以内的
    中断则不受FreeRTOS管理。

-   代码清单:临界段-2_ **(1)-2**\ ：将configMAX_SYSCALL_INTERRUPT_PRIORITY的值写入BASEPRI寄存器，实现关中断（准确来说是关部分中断）。


带返回值的关中断函数
^^^^^^^^^^^^^^^^^^^^^^^^^^

-   代码清单:临界段-2_ **(2)**\ ：带返回值的关中断函数，可以嵌套，可以在中断里面使用。带返回值的意思是：在往BASEPRI写入新的
值的时候，先将BASEPRI的值保存起来，在更新完BASEPRI的值的时候，将之前保存好的BASEPRI的值返回，返回的值作为形参传入开中断函数。

-   代码清单:临界段-2_ **(2)-1**\ ：configMAX_SYSCALL_INTERRUPT_PRIORITY是一个在FreeRTOSConfig.h中定义的宏，即要写入到
    BASEPRI寄存器的值。该宏默认定义为191，高四位有效，即等于0xb0，或者是11，即优先级大于等于11的中断都会被屏蔽，11以内的中断
    则不受FreeRTOS管理

-   代码清单:临界段-2_ **(2)-2**\ ：保存BASEPRI的值，记录当前哪些中断被关闭。

-   代码清单:临界段-2_ **(2)-3**\ ：更新BASEPRI的值。

-   代码清单:临界段-2_ **(2)-4**\ ：返回原来BASEPRI的值

开中断
~~~~~~~~~

FreeRTOS开中断的函数在portmacro.h中定义，具体实现见 代码清单:临界段-3_。

.. code-block:: guess
    :caption: 代码清单:临界段-3开中断函数
    :name: 代码清单:临界段-3
    :linenos:

    /* 不带中断保护的开中断函数 */
    #define portENABLE_INTERRUPTS() vPortSetBASEPRI( 0 )(2)

    /* 带中断保护的开中断函数 */
    #define portCLEAR_INTERRUPT_MASK_FROM_ISR(x) vPortSetBASEPRI(x)(3)

    void vPortSetBASEPRI( uint32_t ulBASEPRI )(1)
    {
        __asm
        {
            msr basepri, ulBASEPRI
        }
    }


-   代码清单:临界段-3_ **(1)**\ ：开中断函数，具体是将传进来的形参更新到BASEPRI寄存器。根据传进来形参的不同，分为中断保护版本与非中断保护版本。

-   代码清单:临界段-3_ **(2)**\ ：不带中断保护的开中断函数，直接将BASEPRI的值设置为0，与portDISABLE_INTERRUPTS()成对使用。

-   代码清单:临界段-3_ **(3)**\ ：带中断保护的开中断函数，将上一次关中断时保存的BASEPRI的值作为形参，与portSET_INTERRUPT_MASK_FROM_ISR()成对使用。

进入/退出临界段的宏
~~~~~~~~~~~~~~~~~~~~

进入和退出临界段的宏在task.h中定义，具体见 代码清单:临界段-4_。

.. code-block:: c
    :caption: 代码清单:临界段-4进入和退出临界段宏定义
    :name: 代码清单:临界段-4
    :linenos:


    #define taskENTER_CRITICAL()		portENTER_CRITICAL()
    #define taskENTER_CRITICAL_FROM_ISR() portSET_INTERRUPT_MASK_FROM_ISR()

    #define taskEXIT_CRITICAL()		portEXIT_CRITICAL()
    #define taskEXIT_CRITICAL_FROM_ISR( x ) portCLEAR_INTERRUPT_MASK_FROM_ISR( x )
进入和退出临界段的宏分中断保护版本和非中断版本，但最终都是通过开/关中断来实现。有关开/光中断的底层代码我们已经讲解，那么接下来的退出和进入临界段的代码配套注释来理解即可。

进入临界段
^^^^^^^^^^

进入临界段，不带中断保护版本且不能嵌套的代码实现具体见 代码清单:临界段-5_。

不带中断保护版本，不能嵌套
'''''''''''''''''''''''''''''''''''

.. code-block:: c
    :caption: 代码清单:临界段-5进入临界段，不带中断保护版本，不能嵌套
    :name: 代码清单:临界段-5
    :linenos:


    /* ==========进入临界段，不带中断保护版本，不能嵌套=============== */
    /* 在task.h中定义 */
    #define taskENTER_CRITICAL()		portENTER_CRITICAL()

    /* 在portmacro.h中定义 */
    #define portENTER_CRITICAL()		vPortEnterCritical()

    /* 在port.c中定义 */
    void vPortEnterCritical( void )
    {
        portDISABLE_INTERRUPTS();
        uxCriticalNesting++;(1)

    if ( uxCriticalNesting == 1 )(2)
        {
    configASSERT( ( portNVIC_INT_CTRL_REG & portVECTACTIVE_MASK ) == 0 );
        }
    }

    /* 在portmacro.h中定义 */
    #define portDISABLE_INTERRUPTS()	vPortRaiseBASEPRI()

    /* 在portmacro.h中定义 */
    static portFORCE_INLINE void vPortRaiseBASEPRI( void )
    {
    uint32_t ulNewBASEPRI = configMAX_SYSCALL_INTERRUPT_PRIORITY;

        __asm
        {
            msr basepri, ulNewBASEPRI
            dsb
            isb
        }
    }

-   代码清单:临界段-5_ **(1)**\：uxCriticalNesting是在port.c中定义的静态变量，表示临界段嵌套计数
    器，默认初始化为0xaaaaaaaa，在调度器启动时会被重新初始化为
    0：vTaskStartScheduler()->xPortStartScheduler()->uxCriticalNesting = 0。

-   代码清单:临界段-5_ **(2)**\ ：如果uxCriticalNesting等于1，即一层嵌套，要确保当前没有中断活跃，
    即内核外设SCB中的中断和控制寄存器SCB_ICSR的低8位要等于0。有关SCB_ICSR的具体描述可参考
    “STM32F10xxx Cortex-M3 programmingmanual-4.4.2小节”。

进入临界段，带中断保护版本且可以嵌套的代码实现具体见 代码清单:临界段-6_。

带中断保护版本，可以嵌套
''''''''''''''''''''''''''''''''

.. code-block:: guess
    :caption: 代码清单:临界段-6进入临界段，带中断保护版本，可以嵌套
    :name: 代码清单:临界段-6
    :linenos:

    /* ==========进入临界段，带中断保护版本，可以嵌套=============== */
    /* 在task.h中定义 */
    #define taskENTER_CRITICAL_FROM_ISR()  portSET_INTERRUPT_MASK_FROM_ISR()

    /* 在portmacro.h中定义 */
    #define portSET_INTERRUPT_MASK_FROM_ISR()		ulPortRaiseBASEPRI()

    /* 在portmacro.h中定义 */
    static portFORCE_INLINE uint32_t ulPortRaiseBASEPRI( void )
    {
    uint32_t ulReturn, ulNewBASEPRI = configMAX_SYSCALL_INTERRUPT_PRIORITY;

        __asm
        {
            mrs ulReturn, basepri
            msr basepri, ulNewBASEPRI
            dsb
            isb
        }

    return ulReturn;
    }



退出临界段
^^^^^^^^^^^^^^^

退出临界段，不带中断保护版本且不能嵌套的代码实现具体见 代码清单:临界段-7_。

不带中断保护的版本，不能嵌套
''''''''''''''''''''''''''''''''''''''

.. code-block:: c
    :caption: 代码清单:临界段-7退出临界段，不带中断保护版本，不能嵌套
    :name: 代码清单:临界段-7
    :linenos:


    /* ==========退出临界段，不带中断保护版本，不能嵌套=============== */
    /* 在task.h中定义 */
    #define taskEXIT_CRITICAL()		portEXIT_CRITICAL()

    /* 在portmacro.h中定义 */
    #define portEXIT_CRITICAL()		vPortExitCritical()

    /* 在port.c中定义 */
    void vPortExitCritical( void )
    {
        configASSERT( uxCriticalNesting );
        uxCriticalNesting--;
    if ( uxCriticalNesting == 0 )
        {
            portENABLE_INTERRUPTS();
        }
    }

    /* 在portmacro.h中定义 */
    #define portENABLE_INTERRUPTS()	vPortSetBASEPRI( 0 )

    /* 在portmacro.h中定义 */
    static portFORCE_INLINE void vPortSetBASEPRI( uint32_t ulBASEPRI )
    {
        __asm
        {
            msr basepri, ulBASEPRI
        }
    }


带中断保护的版本，可以嵌套
''''''''''''''''''''''''''

.. code-block:: guess
    :caption: 代码清单:临界段-8退出临界段，带中断保护版本，可以嵌套
    :name: 代码清单:临界段-8
    :linenos:


    /* ==========退出临界段，带中断保护版本，可以嵌套=============== */
    /* 在task.h中定义 */
    #define taskEXIT_CRITICAL_FROM_ISR( x ) portCLEAR_INTERRUPT_MASK_FROM_ISR( x )

    /* 在portmacro.h中定义 */
    #define portCLEAR_INTERRUPT_MASK_FROM_ISR(x)	vPortSetBASEPRI(x)

    /* 在portmacro.h中定义 */
    static portFORCE_INLINE void vPortSetBASEPRI( uint32_t ulBASEPRI )
    {
        __asm
        {
            msr basepri, ulBASEPRI
        }
    }

临界段代码的应用
~~~~~~~~~~~~~~~~

在FreeRTOS中，对临界段的保护出现在两种场合，一种是在中断场合一种是在非中断场合，具体的应用见 代码清单:临界段-9_。

.. code-block:: c
    :caption: 代码清单:临界段-9临界段代码应用
    :name: 代码清单:临界段-9
    :linenos:


    /* 在中断场合，临界段可以嵌套 */
    {
    uint32_t ulReturn;
    /* 进入临界段，临界段可以嵌套 */
        ulReturn = taskENTER_CRITICAL_FROM_ISR();

    /* 临界段代码 */

    /* 退出临界段 */
        taskEXIT_CRITICAL_FROM_ISR( ulReturn );
    }

    /* 在非中断场合，临界段不能嵌套 */
    {
    /* 进入临界段 */
        taskENTER_CRITICAL();

    /* 临界段代码 */

    /* 退出临界段*/
        taskEXIT_CRITICAL();
    }

实验现象
~~~~~~~~~~~~

本章没有实验，充分理解本章内容即可，这么简单，其实也没啥好理解的。
