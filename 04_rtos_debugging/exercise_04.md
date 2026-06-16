# Experiment 4

# 1️⃣ Task Visibility and Runtime Observation

## Goal of this step
How to observe RTOS behavior, not just assume it.
In this step, we will build a simple RTOS project with:

* 2 tasks
* each task updates its own execution counter
* each task prints:
    * task name
    * execution count
    * current tick count


This gives a clean basis for:

* debugger observation
* runtime reasoning
* later stack/timing analysis

## Add these global counters in USER CODE section

```c
volatile uint32_t g_monitor_count = 0;
volatile uint32_t g_worker_count  = 0;
```

## Add this helper function
```c
void uart_print(char *msg)
{
    HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), HAL_MAX_DELAY);
}
```

## Put this code inside the generated MonitorTask function

```c
void StartMonitorTask(void *argument)
{
    char msg[120];

    for(;;)
    {
        g_monitor_count++;

        snprintf(msg, sizeof(msg),
                 "MonitorTask: count=%lu, tick=%lu, worker_count=%lu\r\n",
                 g_monitor_count,
                 osKernelGetTickCount(),
                 g_worker_count);

        uart_print(msg);

        osDelay(1000);
    }
}
```

## Put this code inside the generated WorkerTask function

```c
void StartWorkerTask(void *argument)
{
    char msg[120];

    for(;;)
    {
        g_worker_count++;

        snprintf(msg, sizeof(msg),
                 "WorkerTask : count=%lu, tick=%lu\r\n",
                 g_worker_count,
                 osKernelGetTickCount());

        uart_print(msg);

        for (volatile uint32_t i = 0; i < 500000; i++)
        {
            // Simulated small workload
        }

        osDelay(300);
    }
}
```

## Observe

On serial terminal, You should see output like:

```
WorkerTask : count=1, tick=5
WorkerTask : count=2, tick=308
WorkerTask : count=3, tick=611
MonitorTask: count=1, tick=1004, worker_count=3
WorkerTask : count=4, tick=914
WorkerTask : count=5, tick=1216
WorkerTask : count=6, tick=1519
MonitorTask: count=2, tick=2005, worker_count=6
```

## What this step demonstrates
This gives participants visible evidence of:

* both tasks are running
* tasks run at different rates
* tick count is increasing
* one task can monitor another task’s progress using shared counters

📌 `Before debugging an RTOS problem, first make task behavior visible.`

📌 `If you cannot observe which task is running, how often it runs, and whether it is progressing, debugging becomes guesswork.`

Understand:

* task execution can be made visible using counters + logs
* RTOS timing can be correlated using tick count
* shared state can help us monitor task health
* this is the first step before deeper debugging

----

# 2️⃣ Debugger-Based Observation of RTOS Tasks

## Goal of this step

How to answer these questions using debugger:

* Is the scheduler running?
* Which task is running right now?
* Are task counters updating?
* Is one task executing more often than another?
* Is the system alive or stuck?

This step is very important because it builds debugging discipline, not just coding skill.

## Add this global variable

```c
volatile uint32_t g_last_task_id = 0;
```

## Update MonitorTask like this

```c
void StartMonitorTask(void *argument)
{
    char msg[120];

    for(;;)
    {
        g_last_task_id = 1;
        g_monitor_count++;

        snprintf(msg, sizeof(msg),
                 "MonitorTask: count=%lu, tick=%lu, worker_count=%lu\r\n",
                 g_monitor_count,
                 osKernelGetTickCount(),
                 g_worker_count);

        uart_print(msg);

        osDelay(1000);
    }
}
```

## Update WorkerTask like this

```c
void StartWorkerTask(void *argument)
{
    char msg[120];

    for(;;)
    {
        g_last_task_id = 2;
        g_worker_count++;

        snprintf(msg, sizeof(msg),
                 "WorkerTask : count=%lu, tick=%lu\r\n",
                 g_worker_count,
                 osKernelGetTickCount());

        uart_print(msg);

        for (volatile uint32_t i = 0; i < 500000; i++)
        {
            // Simulated small workload
        }

        osDelay(300);
    }
}
```

Now in debugger you can watch:

* g_monitor_count
* g_worker_count
* g_last_task_id

So, one can easily infer:

* which task ran recently
* whether both tasks are progressing
* whether scheduler is switching

# Debugging Procedure

## Part A — Start Debug Session
What to do

* Build project
* Flash board
* Click Debug
* Let code enter debugger mode

## Part B — Add Watch Expressions
In Expressions window, add:

* `g_monitor_count`
* `g_worker_count`
* `g_last_task_id`

## Part C — Put Breakpoints

Set breakpoints at these locations:

* Breakpoint 1: Inside MonitorTask, on this line: `g_monitor_count++;`
* Breakpoint 2: Inside WorkerTask, on this line: `g_worker_count++;`

## Part D — Run and Observe

* Press Resume.
* The debugger should stop at one of the breakpoints.
* Now understand:

    * currently stopped task = task whose breakpoint hit
    * global counters show progress
    * values changing over time indicate scheduler is active

* **when breakpoint hits MonitorTask**: The scheduler has selected MonitorTask, and execution has stopped here because this task reached the breakpoint.
* **when breakpoint hits WorkerTask**: Now WorkerTask is executing. Compare this with previous breakpoint hit to understand scheduler switching.

Then compare:

* `g_worker_count` increasing faster
* `g_monitor_count` increasing slower

* WorkerTask runs more frequently because it delays for 300 ms
* MonitorTask runs less frequently because it delays for 1000 ms
* debugger confirms runtime behavior seen on UART
* counters and breakpoints together give reliable insight

📌 `UART print tells you the symptom. Debugger tells you the execution point.`

📌 `For RTOS systems, always combine runtime logs with debugger observation.`

----

# 3️⃣ Task Stack Stress / Runtime Instability Demo

## Goal of this step

In RTOS each task has its own stack, and poor stack sizing can cause erratic or unstable behavior

## What we will do
We will intentionally make WorkerTask use a large local buffer.
This creates stack pressure.
Depending on current stack size and build settings, you may observe:

* strange behavior
* garbled UART
* missed execution
* HardFault / reset
* or no immediate visible failure but unstable system

That is okay — the demo is still valid.

## Keep task stack sizes small enough to make the effect visible
Recommended:

* MonitorTask → 256 words
* WorkerTask → 128 words (important for demo impact)

## Replace WorkerTask with this stack-stress version
````c
void StartWorkerTask(void *argument)
{
    char msg[120];

    for(;;)
    {
        g_last_task_id = 2;
        g_worker_count++;

        /* Intentional large local buffer to stress task stack */
        uint8_t large_buffer[700];

        for (uint32_t i = 0; i < sizeof(large_buffer); i++)
        {
            large_buffer[i] = (uint8_t)(i & 0xFF);
        }

        snprintf(msg, sizeof(msg),
                 "WorkerTask : count=%lu, tick=%lu, buffer_last=%u\r\n",
                 g_worker_count,
                 osKernelGetTickCount(),
                 large_buffer[699]);

        uart_print(msg);

        for (volatile uint32_t i = 0; i < 500000; i++)
        {
            // Simulated workload
        }

        osDelay(300);
    }
}
````
## Observe

Depending on stack size and compiler/debug settings, you may see one of these:
* System still runs, but behaves strangely
* WorkerTask output becomes unreliable
* board HardFault / resets / freezes

What this demonstrates
This teaches a very important embedded RTOS reality:

* each task gets its own stack
* local variables consume task stack
* deep call chains consume task stack
* large arrays inside task functions are dangerous
* stack overflow often appears as random instability

📌 `A task can be logically correct and still fail because of poor stack sizing.`

📌 `In embedded systems, resource sizing is as important as logic correctness.`


common fixes are:
* increase task stack size
* avoid large local buffers
* move large buffers to static/global memory if appropriate
* reduce function nesting
* monitor stack usage in production systems

----

# 4️⃣ Timing Behavior Analysis Using RTOS Tick

## Goal of this step
Teach participants to verify:

* when a task is supposed to run
* when it actually runs
* how to measure task period using RTOS ticks
* why timing correctness matters, not just functional correctness

This is a very important industry concept because many embedded bugs are not “wrong output” bugs — they are wrong timing bugs.

## What we will do
We will create:

* PeriodicTask → intended to run every 500 ms
* LoadTask → introduces extra background workload

Inside PeriodicTask, we will measure:

* current tick
* previous tick
* delta (current_tick - previous_tick)

This lets one to see the actual execution interval.

## CubeMX / CubeIDE configuration
* FreeRTOS: Enable FreeRTOS and create 2 tasks:
    * Task 1
        * Name: PeriodicTask
        * Priority: Normal
        * Stack size: 256 words

    * Task 2
        * Name: LoadTask
        * Priority: Low
        * Stack size: 256 words

## Add these global variables in USER CODE section

````c
volatile uint32_t g_periodic_count = 0;
volatile uint32_t g_load_count = 0;
volatile uint32_t g_last_delta_ticks = 0;
````

## Put this code inside the generated PeriodicTask function

````c
void StartPeriodicTask(void *argument)
{
    char msg[150];
    uint32_t previous_tick = osKernelGetTickCount();
    uint32_t current_tick  = 0;
    uint32_t delta_tick    = 0;

    for(;;)
    {
        current_tick = osKernelGetTickCount();
        delta_tick   = current_tick - previous_tick;
        previous_tick = current_tick;

        g_periodic_count++;
        g_last_delta_ticks = delta_tick;

        snprintf(msg, sizeof(msg),
                 "PeriodicTask: count=%lu, tick=%lu, delta=%lu ticks\r\n",
                 g_periodic_count,
                 current_tick,
                 delta_tick);

        uart_print(msg);

        osDelay(500);
    }
}
````

## Put this code inside the generated LoadTask function

````c
void StartLoadTask(void *argument)
{
    char msg[120];

    for(;;)
    {
        g_load_count++;

        snprintf(msg, sizeof(msg),
                 "LoadTask    : count=%lu, tick=%lu\r\n",
                 g_load_count,
                 osKernelGetTickCount());

        uart_print(msg);

        for (volatile uint32_t i = 0; i < 1200000; i++)
        {
            // Simulated background CPU load
        }

        osDelay(200);
    }
}
````

## Observe

On terminal, You should see output like:

````
PeriodicTask: count=1, tick=5, delta=0 ticks
LoadTask    : count=1, tick=7
LoadTask    : count=2, tick=210
LoadTask    : count=3, tick=413
PeriodicTask: count=2, tick=506, delta=501 ticks
LoadTask    : count=4, tick=616
LoadTask    : count=5, tick=819
PeriodicTask: count=3, tick=1007, delta=501 ticks
````

## What this demonstrates
This is the key learning:

* PeriodicTask intends to run every 500 ms
* measured delta should be approximately 500 ticks
* LoadTask adds background activity
* verify actual timing, not assume it

📌 `In real-time systems, correct logic is not enough. The code must also run at the correct time.`

📌 `If a task is intended to run every 500 ms, we should verify that with data — not assumption.`

----