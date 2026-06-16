# Exercise 3

# 1️⃣ Same Priority vs Different Priority — Basic Scheduler Observation

## Goal of this step
Now we start making the scheduler visible.
In this step, we will create two CPU-active tasks that both print periodically, and we will first run them at the same priority.

This step is only for:

* seeing that both tasks run
* preparing the ground for priority experiments in the next step

We are not doing starvation yet.

## CubeMX / CubeIDE configuration

* UART
    * USART2
    * Asynchronous
    * Baud rate = 115200

* FreeRTOS: Enable FreeRTOS and create 2 tasks:
    * Task 1
        * Name: TaskA
        * Priority: Normal
        * Stack size: 256 words

    * Task 2
        * Name: TaskB
        * Priority: Normal
        * Stack size: 256 words

Then generate code.

For this step, LED is not required. We only want clean scheduler observation through UART.

## Add this helper function in USER CODE section

````c
void uart_print(char *msg)
{
    HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), HAL_MAX_DELAY);
}
````

## Put this code inside the generated TaskA function

````c
void StartTaskA(void *argument)
{
    uint32_t countA = 0;
    char msg[80];

    for(;;)
    {
        countA++;
        snprintf(msg, sizeof(msg), "TaskA: running, count = %lu\r\n", countA);
        uart_print(msg);

        for (volatile uint32_t i = 0; i < 200000; i++)
        {
            // small CPU workload
        }

        osDelay(500);
    }
}
````

## Put this code inside the generated TaskB function

````c
void StartTaskB(void *argument)
{
    uint32_t countB = 0;
    char msg[80];

    for(;;)
    {
        countB++;
        snprintf(msg, sizeof(msg), "TaskB: running, count = %lu\r\n", countB);
        uart_print(msg);

        for (volatile uint32_t i = 0; i < 200000; i++)
        {
            // small CPU workload
        }

        osDelay(500);
    }
}
````

## Observe

On terminal, you should see both tasks running, for example:

```
TaskA: running, count = 1
TaskB: running, count = 1
TaskA: running, count = 2
TaskB: running, count = 2
TaskA: running, count = 3
TaskB: running, count = 3
```

## What this step demonstrates
At this point:

* both tasks have the same priority
* both tasks are getting CPU time
* both tasks perform some small work and then delay
* scheduler is switching between them

This is the clean baseline before we change priorities.

📌 `Since both tasks have the same priority and both periodically block using osDelay(), the scheduler gets chances to run both fairly.`

----

# 2️⃣ Different Priorities — Higher Priority Task Runs First

## Goal of this step
Now we keep the same basic demo, but change task priorities so participants can observe:

* when both tasks become READY together, the higher-priority task gets CPU first

## Change task priorities
Set:

* TaskA → Above Normal
* TaskB → Below Normal

Then Generate Code.

## Replace the body of TaskA with this

````c
void StartTaskA(void *argument)
{
    uint32_t countA = 0;
    char msg[100];

    for(;;)
    {
        countA++;
        snprintf(msg, sizeof(msg), "TaskA (HIGH): running first, count = %lu, tick = %lu\r\n",
                 countA, osKernelGetTickCount());
        uart_print(msg);

        for (volatile uint32_t i = 0; i < 400000; i++)
        {
            // CPU workload
        }

        osDelay(500);
    }
}
``
````

## Replace the body of TaskB with this

```c
void StartTaskB(void *argument)
{
    uint32_t countB = 0;
    char msg[100];

    for(;;)
    {
        countB++;
        snprintf(msg, sizeof(msg), "TaskB (LOW) : running later, count = %lu, tick = %lu\r\n",
                 countB, osKernelGetTickCount());
        uart_print(msg);

        for (volatile uint32_t i = 0; i < 400000; i++)
        {
            // CPU workload
        }

        osDelay(500);
    }
}
```

## Observe
On terminal, you will still see both tasks running, but now the order should become more consistent:

```
TaskA (HIGH): running first, count = 1, tick = 23
TaskB (LOW) : running later, count = 1, tick = 23
TaskA (HIGH): running first, count = 2, tick = 523
TaskB (LOW) : running later, count = 2, tick = 523
TaskA (HIGH): running first, count = 3, tick = 1023
TaskB (LOW) : running later, count = 3, tick = 1023
```

## What this step demonstrates
This shows:

* both tasks are still healthy
* both tasks still get CPU time
* but when both are READY around the same time, higher-priority TaskA runs first

📌 `Priority does not mean lower-priority tasks never run. It means the scheduler prefers the higher-priority READY task.`

📌 `As long as the higher-priority task blocks or delays properly, lower-priority tasks still get execution time.`

----

# 3️⃣ Task Starvation Demo — High Priority Task Blocks Lower Priority Task

## Goal of this step
We will now intentionally create a bad RTOS design:

* TaskA = high priority
* TaskB = low priority
* TaskA will never block / delay

This will show: `if a high-priority task keeps running continuously, the lower-priority task may never get CPU time` This is called **task starvation** .

## Replace the body of TaskA with this
```c
void StartTaskA(void *argument)
{
    uint32_t countA = 0;
    char msg[100];

    for(;;)
    {
        countA++;

        snprintf(msg, sizeof(msg), "TaskA (HIGH): hogging CPU, count = %lu\r\n", countA);
        uart_print(msg);

        for (volatile uint32_t i = 0; i < 800000; i++)
        {
            // Heavy CPU workload
        }

        // IMPORTANT: No osDelay() here
        // This task never voluntarily blocks
    }
}
```

## Replace the body of TaskB with this

```c
void StartTaskB(void *argument)
{
    uint32_t countB = 0;
    char msg[100];

    for(;;)
    {
        countB++;

        snprintf(msg, sizeof(msg), "TaskB (LOW) : should run, count = %lu\r\n", countB);
        uart_print(msg);

        osDelay(500);
    }
}
```

## Observe

On terminal, You should now see mostly or only:
```
TaskA (HIGH): hogging CPU, count = 1
TaskA (HIGH): hogging CPU, count = 2
TaskA (HIGH): hogging CPU, count = 3
TaskA (HIGH): hogging CPU, count = 4
```

You will likely see TaskB output stop completely, or appear extremely rarely depending on exact system behavior.

## What this demonstrates
This is the key learning:

* scheduler always prefers highest-priority READY task
* TaskA is high priority
* TaskA never blocks
* so TaskB gets no chance to run

That is **task starvation**.

📌 `The RTOS is not broken. The design is broken.`

📌 `A high-priority task that never blocks can starve the rest of the system.`

----

# 4️⃣ Fix Starvation — Let Lower Priority Task Run Again

## Goal of this step
Now we fix the bad design from Step 3.
We keep:

* TaskA = higher priority
* TaskB = lower priority

But we modify TaskA so that it periodically yields CPU by blocking briefly. This will show: `higher priority is okay as long as the task blocks appropriately`

## Replace the body of TaskA with this

```c
void StartTaskA(void *argument)
{
    uint32_t countA = 0;
    char msg[100];

    for(;;)
    {
        countA++;

        snprintf(msg, sizeof(msg), "TaskA (HIGH): running, but now yielding, count = %lu\r\n", countA);
        uart_print(msg);

        for (volatile uint32_t i = 0; i < 800000; i++)
        {
            // Heavy CPU workload
        }

        osDelay(1);   // IMPORTANT FIX: allow scheduler to run other tasks
    }
}
```

## Keep / replace the body of TaskB with this

```c
void StartTaskB(void *argument)
{
    uint32_t countB = 0;
    char msg[100];

    for(;;)
    {
        countB++;

        snprintf(msg, sizeof(msg), "TaskB (LOW) : running again, count = %lu\r\n", countB);
        uart_print(msg);

        osDelay(500);
    }
}
```

## Observe

On terminal, Now you should again see both tasks printing, something like:

```
TaskA (HIGH): running, but now yielding, count = 1
TaskA (HIGH): running, but now yielding, count = 2
TaskA (HIGH): running, but now yielding, count = 3
TaskB (LOW) : running again, count = 1
TaskA (HIGH): running, but now yielding, count = 4
TaskA (HIGH): running, but now yielding, count = 5
TaskB (LOW) : running again, count = 2
...
```

TaskA will still appear more often because it has:

* higher priority
* short delay
* continuous workload

But now TaskB is no longer starved.

## What this step demonstrates
This completes the scheduler lesson:

* high priority task can dominate if badly written
* starvation is caused by design, not by RTOS failure
* once the high-priority task blocks properly, lower-priority tasks run again

📌 `Priority alone is not the problem. The real issue is whether a higher-priority task gives the scheduler a chance to run others.`

📌 `In RTOS design, every continuously running task must be carefully reviewed for blocking behavior.`

## Why osDelay(1) is enough here
Even a very small blocking point is enough to move the task from READY to BLOCKED temporarily.
That gives the scheduler a chance to select:

* lower-priority ready tasks
* or other eligible tasks in the system

`Use delay or blocking intelligently based on task responsibility. Periodic tasks should block appropriately. Event-driven tasks should wait on events. Busy looping is usually a design smell.`

## Ways to fix starvation

* Add proper blocking (osDelay, vTaskDelayUntil)
* Wait on queue/semaphore/notification
* Reduce task priority if incorrect
* Break long processing into smaller steps
* Convert polling design to event-driven design

📌 `The goal is not to ‘slow down’ the task. The goal is to make the task behave according to its real responsibility.`