# Exercise 7

# 1️⃣ Three-Stage Embedded Data Pipeline


# Goal of this step

Build a small RTOS pipeline where:

* **ProducerTask** generates raw data
* **ProcessingTask** classifies that data
* **ConsumerTask** prints the processed result

This is no longer “just queue API usage.”  
This is now a **modular embedded system pattern**.

## Why this is different from Exercise 5

**Exercise 5 focus**

* Queue working
* Queue full
* Timeout
* Backpressure
* Producer vs consumer rates

**Exercise 7 focus**

* **Task responsibility separation**
* **Data moving through multiple stages**
* **Producer → processing → output pipeline**
* **Designing a maintainable embedded architecture**

## CubeMX / CubeIDE configuration

* UART
    * Enable:
    * **USART2**
    * Asynchronous
    * **115200 baud**
* FreeRTOS tasks: Create **3 tasks**:
    * Task 1
        * **Name:** `ProducerTask`
        * **Priority:** Normal
        * **Stack size:** 256 words
    * Task 2
        * **Name:** `ProcessingTask`
        * **Priority:** Normal
        * **Stack size:** 256 words
    * Task 3
        * **Name:** `ConsumerTask`
        * **Priority:** Low
        * **Stack size:** 256 words
* Queues: Create **2 queues**:
    * Queue 1
        * **Name:** `RawQueue`
        * **Item size:** `uint32_t`
        * **Length:** `5`
    * Queue 2
        * **Name:** `ProcessedQueue`
        * **Item size:** `uint32_t`
        * **Length:** `5`

Then generate code.

## Data design for this step

To keep the first step simple, we will use this idea:

**ProducerTask**

Generates a raw number like:

* 100
* 350
* 720
* 950
* etc.

**ProcessingTask**

Classifies it into levels:

* **0 = LOW**
* **1 = MID**
* **2 = HIGH**

**ConsumerTask**

Receives classification and prints result.

## Add these globals in `USER CODE BEGIN PTD` or `PV`

```c
volatile uint32_t g_raw_produced_count = 0;
volatile uint32_t g_processed_count    = 0;
volatile uint32_t g_consumed_count     = 0;
```

***

## Keep / add includes

At top:

```c
#include <stdio.h>
#include <string.h>
```

## Add UART helper

```c
void uart_print(char *msg)
{
    HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), HAL_MAX_DELAY);
}
```

## Replace `StartProducerTask()` with this

```c
void StartProducerTask(void *argument)
{
    uint32_t raw_value = 50;
    osStatus_t status;

    for(;;)
    {
        raw_value += 125;

        if (raw_value > 1000)
        {
            raw_value = 50;
        }

        status = osMessageQueuePut(RawQueueHandle, &raw_value, 0, 0);

        if (status == osOK)
        {
            g_raw_produced_count++;
        }

        osDelay(300);
    }
}

```


## Replace `StartProcessingTask()` with this

```c
void StartProcessingTask(void *argument)
{
    uint32_t raw_value = 0;
    uint32_t processed_level = 0;
    osStatus_t status_get;
    osStatus_t status_put;

    for(;;)
    {
        status_get = osMessageQueueGet(RawQueueHandle, &raw_value, NULL, osWaitForever);

        if (status_get == osOK)
        {
            if (raw_value < 300)
            {
                processed_level = 0;   // LOW
            }
            else if (raw_value < 700)
            {
                processed_level = 1;   // MID
            }
            else
            {
                processed_level = 2;   // HIGH
            }

            status_put = osMessageQueuePut(ProcessedQueueHandle, &processed_level, 0, 0);

            if (status_put == osOK)
            {
                g_processed_count++;
            }
        }
    }
}
```

## Replace `StartConsumerTask()` with this

```c
void StartConsumerTask(void *argument)
{
    uint32_t processed_level = 0;
    osStatus_t status;
    char msg[180];
    const char *level_text = "UNKNOWN";
    uint32_t raw_q_count = 0;
    uint32_t processed_q_count = 0;

    for(;;)
    {
        status = osMessageQueueGet(ProcessedQueueHandle, &processed_level, NULL, osWaitForever);

        if (status == osOK)
        {
            g_consumed_count++;

            if (processed_level == 0)
            {
                level_text = "LOW";
            }
            else if (processed_level == 1)
            {
                level_text = "MID";
            }
            else if (processed_level == 2)
            {
                level_text = "HIGH";
            }

            raw_q_count = osMessageQueueGetCount(RawQueueHandle);
            processed_q_count = osMessageQueueGetCount(ProcessedQueueHandle);

            snprintf(msg, sizeof(msg),
                     "ConsumerTask: level=%s, produced=%lu, processed=%lu, consumed=%lu, raw_q=%lu, proc_q=%lu\r\n",
                     level_text,
                     g_raw_produced_count,
                     g_processed_count,
                     g_consumed_count,
                     raw_q_count,
                     processed_q_count);

            uart_print(msg);

            osDelay(100);
        }
    }
}
```

## What you should observe

You should see output like:

```text
ConsumerTask: level=LOW, produced=1, processed=1, consumed=1, raw_q=0, proc_q=0
ConsumerTask: level=MID, produced=2, processed=2, consumed=2, raw_q=0, proc_q=0
ConsumerTask: level=MID, produced=3, processed=3, consumed=3, raw_q=0, proc_q=0
ConsumerTask: level=HIGH, produced=4, processed=4, consumed=4, raw_q=0, proc_q=0
```

## What this step demonstrates

**ProducerTask**

Responsible only for **data generation**

**ProcessingTask**

Responsible only for **data interpretation / transformation**

**ConsumerTask**

Responsible only for *output / reporting*

This is exactly how you want people to think about embedded software architecture.

📌
> Queues are not the architecture. They are the connectors. The architecture is the separation of system responsibilities into acquisition, processing, and output stages.

> This is how embedded pipelines are built in real systems — sensor acquisition, computation, and communication are separated into stages.

----

# 2️⃣ Add Pipeline Health / Monitor Task

## Goal of this step

Now we make the pipeline more **system-oriented** by adding a **MonitorTask**.

This is important because in industry, a system is not considered complete just because tasks work.  
We also want to know:

* Is the pipeline healthy?
* Are all stages progressing?
* Are queues building up?
* Is one stage becoming a bottleneck?

So this step adds **observability** without changing the core pipeline logic.

## Why this step is important

This step makes Exercise 7 feel much more like a **real embedded system**.

You now move from:

*Tasks are running* to *System health can be observed and diagnosed*

## What this step will do

We will add a fourth task:

## `MonitorTask`

It will periodically print:

* `g_raw_produced_count`
* `g_processed_count`
* `g_consumed_count`
* raw queue occupancy
* processed queue occupancy

This gives a pipeline health snapshot every few seconds.

## What to configure in CubeMX

In your existing **Exercise 7 project**, add one more task:

* Task 4
    * **Name:** `MonitorTask`
    * **Priority:** Low
    * **Stack size:** 256 words

Then generate code.

## Do not change existing Producer / Processing / Consumer tasks

Keep:

* `StartProducerTask()`
* `StartProcessingTask()`
* `StartConsumerTask()`

exactly as they are in your working **7.1**.

## Add `StartMonitorTask()` function

```c
void StartMonitorTask(void *argument)
{
    char msg[200];
    uint32_t raw_q_count = 0;
    uint32_t proc_q_count = 0;

    for(;;)
    {
        raw_q_count = osMessageQueueGetCount(RawQueueHandle);
        proc_q_count = osMessageQueueGetCount(ProcessedQueueHandle);

        snprintf(msg, sizeof(msg),
                 "MonitorTask : produced=%lu, processed=%lu, consumed=%lu, raw_q=%lu, proc_q=%lu\r\n",
                 g_raw_produced_count,
                 g_processed_count,
                 g_consumed_count,
                 raw_q_count,
                 proc_q_count);

        uart_print(msg);

        osDelay(2000);
    }
}
```

## What you should observe

Now along with ConsumerTask output, every 2 seconds you should also see a periodic health snapshot like:

```text
ConsumerTask: level=LOW, produced=1, processed=1, consumed=1, raw_q=0, proc_q=0
ConsumerTask: level=MID, produced=2, processed=2, consumed=2, raw_q=0, proc_q=0
ConsumerTask: level=HIGH, produced=3, processed=3, consumed=3, raw_q=0, proc_q=0
MonitorTask : produced=6, processed=6, consumed=6, raw_q=0, proc_q=0
```

If the pipeline is healthy, counts should remain close.

## What this step demonstrates

This is the key learning of **7.2**:

* a system can be **architecturally correct**
* but we also need **runtime visibility**
* monitor tasks are useful for:
  * health reporting
  * observing bottlenecks
  * debugging field issues
  * validating whether stages are progressing

This is a very industry-relevant idea.

📌
> A good embedded design is not only modular; it is also observable.

> Monitor tasks help us understand whether data is flowing through the pipeline correctly and whether any stage is falling behind.

----

Excellent ✅  
That means **Exercise 7 – Step 2** is now working, and you have:

* **ProducerTask** generating data
* **ProcessingTask** transforming it
* **ConsumerTask** consuming it
* **MonitorTask** reporting health

So now the pipeline is behaving like a **small embedded system**, not just isolated tasks.

***

# 3️⃣ Create a Bottleneck and Observe Which Stage Becomes the Problem

## Goal of this step

Now we deliberately make **one stage slower** and use the already added **MonitorTask** to observe:

* which queue starts growing
* where backlog forms
* whether the rest of the pipeline can keep up

This is a very strong **system-diagnosis exercise**.

## Why this step is important

In industry, the problem is often **not**: `Does the system work?`

The real question is: `Which stage is becoming the bottleneck?`

## What we will do

We will intentionally slow down: **ProcessingTask**

So now the pipeline becomes:

```text
ProducerTask  →  RawQueue  →  ProcessingTask (slow)  →  ProcessedQueue  →  ConsumerTask
```

Because ProcessingTask becomes slow:

* `RawQueue` should start filling
* `ProcessedQueue` may stay low
* Producer may eventually see **RawQueue full**
* MonitorTask will reveal the imbalance

##  Only one code change needed, Replace `StartProcessingTask()` with this slower version

```c
void StartProcessingTask(void *argument)
{
    uint32_t raw_value = 0;
    uint32_t processed_level = 0;
    osStatus_t status_get;
    osStatus_t status_put;

    for(;;)
    {
        status_get = osMessageQueueGet(RawQueueHandle, &raw_value, NULL, osWaitForever);

        if (status_get == osOK)
        {
            /* Simulate heavier processing load */
            osDelay(700);

            if (raw_value < 300)
            {
                processed_level = 0;   // LOW
            }
            else if (raw_value < 700)
            {
                processed_level = 1;   // MID
            }
            else
            {
                processed_level = 2;   // HIGH
            }

            status_put = osMessageQueuePut(ProcessedQueueHandle, &processed_level, 0, 0);

            if (status_put == osOK)
            {
                g_processed_count++;
            }
        }
    }
}
```

***

# Keep everything else unchanged

Do **not** change:

* `StartProducerTask()`
* `StartConsumerTask()`
* `StartMonitorTask()`

Let only the **ProcessingTask** become slower.

## What you should expect now

You should start seeing behavior like this:

```text
MonitorTask : produced=15, processed=6, consumed=6, raw_q=5, proc_q=0
MonitorTask : produced=22, processed=9, consumed=9, raw_q=5, proc_q=0
MonitorTask : produced=29, processed=12, consumed=12, raw_q=5, proc_q=0
```

And possibly ProducerTask may eventually begin failing to send if `RawQueue` gets full, depending on your current Producer behavior.


## What this demonstrates

Since ProcessingTask is slow:

* **RawQueue grows**
* **ProcessedQueue stays low**
* Consumer is not the bottleneck
* Processing stage is clearly the bottleneck

This is exactly the kind of systems reasoning you want participants to develop.


> “When a pipeline becomes unbalanced, the queue before the slow stage begins to fill.”

> “Queue growth tells us where the bottleneck is.”

That is a very strong and practical system design lesson.

***

## How to explain the observation

If `raw_q` grows and `proc_q` stays low, then:

* Producer is faster than ProcessingTask
* ProcessingTask is the constrained stage
* Consumer is probably fine
* the problem is **before** the processing stage, not after it

This is a very useful diagnostic principle.

----