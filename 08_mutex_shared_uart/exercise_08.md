# Exercise 8

# 1️⃣ Shared UART Without Mutex — Observe Corrupted / Interleaved Output

## Goal of this step

Before introducing mutex, participants should first **see the problem**.

So in this step, we intentionally create a system where:

* **TaskA** prints a multi-line message
* **TaskB** prints another multi-line message
* both use the **same UART**
* **no mutex is used**

This should produce:

* interleaved output
* mixed logs
* difficult-to-read terminal output

That creates the **need for mutex**.

## CubeMX / CubeIDE configuration

* UART
    * Enable:
    * **USART2**
    * Asynchronous
    * **115200 baud**
* FreeRTOS tasks
    * Create **2 tasks**:
    * Task 1
        * **Name:** `TaskA`
        * **Priority:** Normal
        * **Stack size:** 256 words
    * Task 2
        * **Name:** `TaskB`
        * **Priority:** Normal
        * **Stack size:** 256 words

Then generate code.


## What this step will do

* TaskA: Prints a 3-line “packet”
* TaskA: Prints a 3-line “packet”

Since both are printing to the same UART with no protection, the terminal output should become mixed.

# Add includes at top

```c
#include <stdio.h>
#include <string.h>
```

## Add UART helper

Use this exact helper:

```c
void uart_print(char *msg)
{
    HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), HAL_MAX_DELAY);
}
```

## Add these globals

```c
volatile uint32_t g_taskA_count = 0;
volatile uint32_t g_taskB_count = 0;
```

## Replace `StartTaskA()` with this

```c
void StartTaskA(void *argument)
{
    char msg[100];

    for(;;)
    {
        g_taskA_count++;

        snprintf(msg, sizeof(msg), "\r\n--- TaskA PACKET START (%lu) ---\r\n", g_taskA_count);
        uart_print(msg);
        osDelay(10);

        snprintf(msg, sizeof(msg), "TaskA DATA1 : value=%lu\r\n", g_taskA_count * 10);
        uart_print(msg);
        osDelay(10);

        snprintf(msg, sizeof(msg), "TaskA DATA2 : value=%lu\r\n", g_taskA_count * 20);
        uart_print(msg);
        osDelay(10);

        snprintf(msg, sizeof(msg), "--- TaskA PACKET END (%lu) ---\r\n", g_taskA_count);
        uart_print(msg);

        osDelay(200);
    }
}
```

## Replace `StartTaskB()` with this

```c
void StartTaskB(void *argument)
{
    char msg[100];

    for(;;)
    {
        g_taskB_count++;

        snprintf(msg, sizeof(msg), "\r\n*** TaskB PACKET START (%lu) ***\r\n", g_taskB_count);
        uart_print(msg);
        osDelay(10);

        snprintf(msg, sizeof(msg), "TaskB STAT1 : code=%lu\r\n", 100 + g_taskB_count);
        uart_print(msg);
        osDelay(10);

        snprintf(msg, sizeof(msg), "TaskB STAT2 : code=%lu\r\n", 200 + g_taskB_count);
        uart_print(msg);
        osDelay(10);

        snprintf(msg, sizeof(msg), "*** TaskB PACKET END (%lu) ***\r\n", g_taskB_count);
        uart_print(msg);

        osDelay(200);
    }
}
```

## What you should observe

You may sometimes see clean output, but quite often the output will become mixed or appear interleaved like:

```text
*** TaskB PACKET START (208) ***
TaskA DATA1 : value=2080
TaskB STAT2 : code=408
--- TaskA PACKET END (208) ---
```

Or other mixed sequences.

## Important point

Even if each individual line prints correctly, the **logical packet** from TaskA or TaskB gets split and mixed with the other task’s output.

That is the actual problem.

## What this step demonstrates

This shows clearly:

* UART is a **shared resource**
* multiple tasks can access it concurrently
* without coordination, output becomes interleaved
* debugging and logging becomes unreliable

That is the real motivation for mutex.

📌
> “The problem is not that UART is broken. The problem is that multiple tasks are accessing the same shared resource without ownership control.”

> “When access to a shared resource must be exclusive, we need a mutex.”

----

# 2️⃣ Protect UART with Mutex — Keep Each Packet Together

## Goal of this step

Now we fix the shared resource problem by using a **mutex**.

So instead of:

* TaskA printing line-by-line independently
* TaskB printing line-by-line independently

we will make sure:

> **one task owns UART for the entire packet**

Only after the full packet is printed will the other task get access.

## What this demonstrates

This is the key mutex lesson:

* **Without mutex** → interleaved packets
* **With mutex** → each packet stays intact

That makes the need for mutex obvious and practical.

## What to configure first in CubeMX

* Add one Mutex
    * Create:
    * **Name:** `UartMutex`

Then generate code.

You should get something like:

```c
osMutexId_t UartMutexHandle;
const osMutexAttr_t UartMutex_attributes = {
  .name = "UartMutex"
};
```

and in `main()`:

```c
UartMutexHandle = osMutexNew(&UartMutex_attributes);
```

## Keep your `uart_print()` helper **simple**

Do **not** put mutex inside `uart_print()` for this step.

Keep it like this:

```c
void uart_print(char *msg)
{
    HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), HAL_MAX_DELAY);
}
```

### Why?

Because in this exercise we want to protect the **whole packet**, not each line individually.

If we lock inside `uart_print()`, only each line becomes atomic — packet can still be interleaved line-by-line.


## Replace `StartTaskA()` with this mutex-protected packet version

```c
void StartTaskA(void *argument)
{
    char msg[100];

    for(;;)
    {
        g_taskA_count++;

        osMutexAcquire(UartMutexHandle, osWaitForever);

        snprintf(msg, sizeof(msg), "\r\n--- TaskA PACKET START (%lu) ---\r\n", g_taskA_count);
        uart_print(msg);
        osDelay(10);

        snprintf(msg, sizeof(msg), "TaskA DATA1 : value=%lu\r\n", g_taskA_count * 10);
        uart_print(msg);
        osDelay(10);

        snprintf(msg, sizeof(msg), "TaskA DATA2 : value=%lu\r\n", g_taskA_count * 20);
        uart_print(msg);
        osDelay(10);

        snprintf(msg, sizeof(msg), "--- TaskA PACKET END (%lu) ---\r\n", g_taskA_count);
        uart_print(msg);

        osMutexRelease(UartMutexHandle);

        osDelay(200);
    }
}
```


## Replace `StartTaskB()` with this mutex-protected packet version

```c
void StartTaskB(void *argument)
{
    char msg[100];

    for(;;)
    {
        g_taskB_count++;

        osMutexAcquire(UartMutexHandle, osWaitForever);

        snprintf(msg, sizeof(msg), "\r\n*** TaskB PACKET START (%lu) ***\r\n", g_taskB_count);
        uart_print(msg);
        osDelay(10);

        snprintf(msg, sizeof(msg), "TaskB STAT1 : code=%lu\r\n", 100 + g_taskB_count);
        uart_print(msg);
        osDelay(10);

        snprintf(msg, sizeof(msg), "TaskB STAT2 : code=%lu\r\n", 200 + g_taskB_count);
        uart_print(msg);
        osDelay(10);

        snprintf(msg, sizeof(msg), "*** TaskB PACKET END (%lu) ***\r\n", g_taskB_count);
        uart_print(msg);

        osMutexRelease(UartMutexHandle);

        osDelay(200);
    }
}
```

## What you should observe now

Before mutex, packets were mixed.

After mutex, output should become clean like this:

```text
--- TaskA PACKET START (1) ---
TaskA DATA1 : value=10
TaskA DATA2 : value=20
--- TaskA PACKET END (1) ---

*** TaskB PACKET START (1) ***
TaskB STAT1 : code=101
TaskB STAT2 : code=201
*** TaskB PACKET END (1) ***
```

Then again:

```text
--- TaskA PACKET START (2) ---
TaskA DATA1 : value=20
TaskA DATA2 : value=40
--- TaskA PACKET END (2) ---

*** TaskB PACKET START (2) ***
TaskB STAT1 : code=102
TaskB STAT2 : code=202
*** TaskB PACKET END (2) ***
```

### Important point

Now each **logical packet stays intact**.

## Why this works

Because now:

* TaskA acquires UART mutex
* TaskB cannot print until TaskA releases it
* TaskA finishes full packet first
* then TaskB gets its turn

So mutex is not making UART “faster.”

It is enforcing: **exclusive ownership of a shared resource**

That is the key idea.

📌
> “A mutex is used when a shared resource must be owned by only one task at a time.”

> “Semaphore is for signaling. Mutex is for ownership.”

----

Perfect — now we do **Exercise 8 – Step 3 only**.

***

# 3️⃣ Bad Practice: Holding a Mutex Too Long

## Goal of this step

Now that participants have seen that mutex **solves interleaved UART access**, we need to teach the equally important next lesson:

> **A mutex should be held only for the minimum required time.**

If a task holds a mutex for too long, then:

* other tasks needing the same resource must wait
* responsiveness drops
* throughput suffers
* whole system behavior can degrade

This is a very important **industry nuance**.

## What we will do

We will intentionally make **TaskA** behave badly:

* TaskA acquires the UART mutex
* prints its packet
* then **keeps the mutex unnecessarily**
* only after an extra delay does it release the mutex

Meanwhile:

* TaskB also wants the UART
* but now it has to wait much longer

So you will observe:

> output is still “correct”  
> but system fairness / responsiveness is worse

That is the exact lesson.

***

## Important distinction

* Step 2 showed: **why mutex is needed**
* Step 3 shows: **how mutex can be misused**

That is what makes this module strong.

##  Only one code change now

We will modify **TaskA only**. Replace `StartTaskA()` with this

```c
void StartTaskA(void *argument)
{
    char msg[100];

    for(;;)
    {
        g_taskA_count++;

        osMutexAcquire(UartMutexHandle, osWaitForever);

        snprintf(msg, sizeof(msg), "\r\n--- TaskA PACKET START (%lu) ---\r\n", g_taskA_count);
        uart_print(msg);
        osDelay(10);

        snprintf(msg, sizeof(msg), "TaskA DATA1 : value=%lu\r\n", g_taskA_count * 10);
        uart_print(msg);
        osDelay(10);

        snprintf(msg, sizeof(msg), "TaskA DATA2 : value=%lu\r\n", g_taskA_count * 20);
        uart_print(msg);
        osDelay(10);

        snprintf(msg, sizeof(msg), "--- TaskA PACKET END (%lu) ---\r\n", g_taskA_count);
        uart_print(msg);

        /* BAD PRACTICE: holding mutex longer than needed */
        osDelay(300);

        osMutexRelease(UartMutexHandle);

        osDelay(200);
    }
}
```

## Keep `StartTaskB()` exactly as in Step 2

Do **not** change TaskB.

Keep it like:

```c
void StartTaskB(void *argument)
{
    char msg[100];

    for(;;)
    {
        g_taskB_count++;

        osMutexAcquire(UartMutexHandle, osWaitForever);

        snprintf(msg, sizeof(msg), "\r\n*** TaskB PACKET START (%lu) ***\r\n", g_taskB_count);
        uart_print(msg);
        osDelay(10);

        snprintf(msg, sizeof(msg), "TaskB STAT1 : code=%lu\r\n", 100 + g_taskB_count);
        uart_print(msg);
        osDelay(10);

        snprintf(msg, sizeof(msg), "TaskB STAT2 : code=%lu\r\n", 200 + g_taskB_count);
        uart_print(msg);
        osDelay(10);

        snprintf(msg, sizeof(msg), "*** TaskB PACKET END (%lu) ***\r\n", g_taskB_count);
        uart_print(msg);

        osMutexRelease(UartMutexHandle);

        osDelay(200);
    }
}
```

## What you should observe

Output should remain **non-interleaved** (mutex still protects UART), but you should notice:

* TaskA packets appear
* TaskB gets delayed noticeably
* overall alternation becomes less smooth
* TaskA seems to “dominate” the UART

Typical behavior may look like:

```text
--- TaskA PACKET START (1) ---
TaskA DATA1 : value=10
TaskA DATA2 : value=20
--- TaskA PACKET END (1) ---

*** TaskB PACKET START (1) ***
TaskB STAT1 : code=101
TaskB STAT2 : code=201
*** TaskB PACKET END (1) ***
```

but with larger gaps before TaskB gets access.

## What this demonstrates

This is the key learning:

* mutex protection is necessary 
* but holding a mutex longer than needed is bad

Because now:

* the critical section includes unnecessary delay
* TaskB is blocked even though UART is not actively being used during that delay
* resource ownership becomes wasteful

That is a classic real-world mistake.

> “A mutex should protect only the true critical section, not unrelated work.”

> “Correctness is not enough. We must also design for responsiveness.”

That is a very strong embedded systems message.

## Why this matters in real systems

This maps directly to real product bugs:

* one task holds a communication lock too long
* logger holds resource while formatting data
* a task acquires mutex before doing computation instead of only around actual resource access
* other tasks get blocked unnecessarily

Result:

* sluggish system
* poor responsiveness
* lower throughput
* hard-to-explain delays

> **Mutex must be used narrowly and intentionally.**

Not:
* too late
* not too early
* and definitely not around unrelated delays or computation

----