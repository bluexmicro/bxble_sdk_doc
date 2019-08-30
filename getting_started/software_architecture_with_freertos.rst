Software Architecture with FreeRTOS
======================================

After the loading process of user image, the system runs to main(). In main(), necessary modules are initialized to work appropriately, then FreeRTOS starts.

There are four FreeRTOS tasks in this system by default.

===== ================ ===========   =========  ==================================
Task  Description      Created by    Priority   Scheduled when  
===== ================ ===========   =========  ==================================
Idle  System Idle      RTOS kernel    Lowest    No other tasks are in Ready state
Timer Timer Daemon     RTOS kernel    Highest   A RTOS timer expires or the Timer Queue receives a message which is going to manipulate a timer
Ble   BLE Stack        SDK code       Highest   BLE Queue receives a message
App   User Application SDK code       Middle    APP Queue receives a message
===== ================ ===========   =========  ==================================

There are two queues used for task scheduling between Ble task and App task.

======== =====================  ========================
Queue     Receive messages in     Messages are sent from
======== =====================  ========================
ble_q     Ble Task               App Task/Ble MAC Isr/HWECC Isr
app_q     App Task               Ble Task(AHI)/Any Context(ASYNC_CALL)
======== =====================  ========================

After the scheduler of FreeRTOS runs, Ble task initializes BLE software protocol stack and hardware, then it try to receive messages from ble_q in a infinite loop. Ble task will go into block state if no message in ble_q. When Ble task is blocked, App task with relatively lower priority will run, and then receives messages in a infinite loop as well from app_q.  

After Ble task succeeds in initialization, a message indicating that BLE protocol stack is ready is generated and sent to app_q. So App task will handle this message once it starts to run. Typically, App task sends GAPM_RESET command to ble_q to continue the application flow. As a response, App task will receive GAPM_CMP_EVT(GAPM_RESET) message.