# Exercise 5

# 1️⃣ Basic Queue Communication using Simulated Sensor Data

## Goal of this step

Show the most important RTOS design idea:

> one task can **produce data**,  
> another task can **consume data**,  
> and the queue acts as a **safe buffer between them**

***

## What this demo will do

We will create:

* **ProducerTask**
  * generates increasing “sensor” values
  * sends those values to a queue every 500 ms

* **ConsumerTask**
  * receives values from the queue
  * prints them using `printf()` (or SWV if you already enabled it)

This is the cleanest possible first queue demo.

## CubeMX / CubeIDE configuration

* UART
    * USART2 @ 115200

* FreeRTOS
    Enable FreeRTOS and create:

    * Task 1

        * **Name:** `ProducerTask`
        * **Priority:** Normal
        * **Stack size:** 256 words

    * Task 2

        * **Name:** `ConsumerTask`
        * **Priority:** Normal
        * **Stack size:** 256 words

    * Add a Queue

        * **Name:** `SensorQueue`
        * **Item size:** `uint32_t`
        * **Length:** `5`

Then generate code.


## If you are using UART-based print, keep this declaration and helper

```c
extern UART_HandleTypeDef huart2;

void uart_print(char *msg)
{
    HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), HAL_MAX_DELAY);
}
```

## Add these global variables in USER CODE section

```c
volatile uint32_t g_produced_count = 0;
volatile uint32_t g_consumed_count = 0;
```
## Replace the body of **ProducerTask** with this

```c
void StartProducerTask(void *argument)
{
    uint32_t sensor_value = 100;
    osStatus_t status;
    char msg[120];

    for(;;)
    {
        sensor_value += 10;

        status = osMessageQueuePut(SensorQueueHandle, &sensor_value, 0, 0);


        if (status == osOK)
        {
            g_produced_count++;

            snprintf(msg, sizeof(msg),
                     "ProducerTask: sent=%lu, produced=%lu, consumed=%lu\r\n",
                     sensor_value,
                     g_produced_count,
                     g_consumed_count);

            uart_print(msg);
        }
        else
        {
            snprintf(msg, sizeof(msg),
                     "ProducerTask: queue full, value=%lu, produced=%lu, consumed=%lu\r\n",
                     sensor_value,
                     g_produced_count,
                     g_consumed_count);

            uart_print(msg);
        }


        osDelay(500);
    }
}
```

***

## Replace the body of **ConsumerTask** with this

```c
void StartConsumerTask(void *argument)
{
    uint32_t received_value = 0;
    osStatus_t status;

    for(;;)
    {
        status = osMessageQueueGet(SensorQueueHandle, &received_value, NULL, osWaitForever);

        if (status == osOK)
        {
            g_consumed_count++;
        }
    }
}
```

## What you should observe

On terminal / SWV console, you should see:

```text
ProducerTask: sent=1070, produced=97, consumed=96
ProducerTask: sent=1080, produced=98, consumed=97
ProducerTask: sent=1090, produced=99, consumed=98
```

***

## What this step demonstrates

This teaches the fundamental queue idea:

* Producer does **not** directly call Consumer
* Consumer does **not** continuously poll shared globals
* Queue safely transfers data between tasks
* `ConsumerTask` blocks on queue until data arrives

📌 `The queue is not just a storage buffer; it is also a synchronization mechanism because the consumer can block until data is available.`

📌 `This is much cleaner than using shared global variables with manual flags.`

---

# 2️⃣ Fast Producer + Slow Consumer + Queue Fill / Backpressure Demo

## Goal of this step

Now we make this more realistic and more interesting.

We will intentionally create:

* **ProducerTask faster**
* **ConsumerTask slower**

So one can observe:

> when producer generates data faster than consumer processes it,  
> the queue starts filling and eventually becomes full

This is a very important **industry behavior**:

* sensor data may arrive faster than it is processed
* communication task may lag
* queue depth becomes a design parameter


## Update `StartProducerTask()` to this

```c
void StartProducerTask(void *argument)
{
    uint32_t sensor_value = 100;
    osStatus_t status;
    char msg[140];

    for(;;)
    {
        sensor_value += 10;

        status = osMessageQueuePut(SensorQueueHandle, &sensor_value, 0, 0);

        if (status == osOK)
        {
            g_produced_count++;

            snprintf(msg, sizeof(msg),
                     "ProducerTask: sent=%lu, produced=%lu, consumed=%lu\r\n",
                     sensor_value,
                     g_produced_count,
                     g_consumed_count);

            uart_print(msg);
        }
        else
        {
            snprintf(msg, sizeof(msg),
                     "ProducerTask: QUEUE FULL, value=%lu, produced=%lu, consumed=%lu\r\n",
                     sensor_value,
                     g_produced_count,
                     g_consumed_count);

            uart_print(msg);
        }

        osDelay(100);   // Faster producer
    }
}
```

***

## Update `StartConsumerTask()` to this

```c
void StartConsumerTask(void *argument)
{
    uint32_t received_value = 0;
    osStatus_t status;
    char msg[140];

    for(;;)
    {
        status = osMessageQueueGet(SensorQueueHandle, &received_value, NULL, osWaitForever);

        if (status == osOK)
        {
            g_consumed_count++;

            snprintf(msg, sizeof(msg),
                     "ConsumerTask: got=%lu, produced=%lu, consumed=%lu\r\n",
                     received_value,
                     g_produced_count,
                     g_consumed_count);

            uart_print(msg);

            osDelay(500);   // Slower consumer
        }
    }
}
```

***

## What this step changes

**Earlier**

* Producer and consumer were roughly balanced

**Now**

* Producer runs every **100 ms**
* Consumer processes one item every **500 ms**

So naturally:

* queue begins to fill
* after queue length limit is reached, producer starts failing to send
* you should begin seeing:

```text
ProducerTask: QUEUE FULL, value=...
```

***

## What you should observe

You should now get a pattern like:

```text
ProducerTask: sent=110, produced=1, consumed=0
ConsumerTask: got=110, produced=1, consumed=1
ProducerTask: sent=120, produced=2, consumed=1
ProducerTask: sent=130, produced=3, consumed=1
ProducerTask: sent=140, produced=4, consumed=1
ProducerTask: sent=150, produced=5, consumed=1
ProducerTask: sent=160, produced=6, consumed=1
ProducerTask: QUEUE FULL, value=170, produced=6, consumed=1
```

Then as consumer drains one item, producer may succeed again briefly.

***

## What this demonstrates

This is a very important RTOS/system-design lesson:

* a queue is not infinite
* task rates matter
* if producer is faster than consumer, backlog builds up
* queue capacity and processing rate must be designed carefully


📌 `Queues decouple tasks, but they do not eliminate system capacity limits.`

📌 `If data is produced faster than it is consumed, the queue becomes a bottleneck.`

---

# 3️⃣ Queue with Timeout + Smarter Producer Behavior

## Goal of this step

In Step 2, producer was using:

```c
osMessageQueuePut(..., 0, 0)
```

So it would **fail immediately** if queue was full.

Now we move one level closer to industry behavior:

> Producer will wait briefly for queue space before giving up.

This helps participants understand:

* immediate fail vs timed wait
* backpressure handling
* producer behavior under congestion


## What we will change

**Earlier**

* Producer tried to send
* If queue full → immediate failure

**Now**

* Producer will wait up to **200 ms**
* If consumer frees space in that time → send succeeds
* Otherwise → producer reports timeout/failure

This is a very good intermediate design before later advanced policies.

## Replace `StartProducerTask()` with this

```c
void StartProducerTask(void *argument)
{
    uint32_t sensor_value = 100;
    osStatus_t status;
    char msg[160];

    for(;;)
    {
        sensor_value += 10;

        /* Wait up to 200 ms for queue space */
        status = osMessageQueuePut(SensorQueueHandle, &sensor_value, 0, 200);

        if (status == osOK)
        {
            g_produced_count++;

            snprintf(msg, sizeof(msg),
                     "ProducerTask: sent=%lu, produced=%lu, consumed=%lu\r\n",
                     sensor_value,
                     g_produced_count,
                     g_consumed_count);

            uart_print(msg);
        }
        else if (status == osErrorTimeout)
        {
            snprintf(msg, sizeof(msg),
                     "ProducerTask: TIMEOUT waiting for queue space, value=%lu, produced=%lu, consumed=%lu\r\n",
                     sensor_value,
                     g_produced_count,
                     g_consumed_count);

            uart_print(msg);
        }
        else
        {
            snprintf(msg, sizeof(msg),
                     "ProducerTask: queue put failed, status=%d, value=%lu\r\n",
                     (int)status,
                     sensor_value);

            uart_print(msg);
        }

        osDelay(100);   // Producer still faster than consumer
    }
}
```

***

## Keep `StartConsumerTask()` like this

```c
void StartConsumerTask(void *argument)
{
    uint32_t received_value = 0;
    osStatus_t status;
    char msg[140];

    for(;;)
    {
        status = osMessageQueueGet(SensorQueueHandle, &received_value, NULL, osWaitForever);

        if (status == osOK)
        {
            g_consumed_count++;

            snprintf(msg, sizeof(msg),
                     "ConsumerTask: got=%lu, produced=%lu, consumed=%lu\r\n",
                     received_value,
                     g_produced_count,
                     g_consumed_count);

            uart_print(msg);

            osDelay(500);   // Slow consumer
        }
    }
}
```

***

## What you should observe

Now instead of immediate:

```text
QUEUE FULL
```

you should often see a mix like:

```text
ProducerTask: sent=110, produced=1, consumed=0
ConsumerTask: got=110, produced=1, consumed=1
ProducerTask: sent=120, produced=2, consumed=1
ProducerTask: sent=130, produced=3, consumed=1
ProducerTask: sent=140, produced=4, consumed=1
ProducerTask: TIMEOUT waiting for queue space, value=150, produced=4, consumed=1
ConsumerTask: got=120, produced=4, consumed=2
ProducerTask: sent=160, produced=5, consumed=2
```

***

# What this demonstrates

This is the key learning:

* a producer does not always need to fail immediately
* sometimes it is better to wait briefly for queue availability
* timeout is a design choice
* queue behavior is part of system flow control

***

# Important teaching point for delivery

Say this clearly:

📌 `Queue full does not always mean data must be dropped immediately. A task may wait for bounded time if the application can tolerate it.`

📌 `Timeout-based queue access is a practical way to balance responsiveness and reliability.`



## Why this has industry flavor

This maps to real systems like:

* sensor producer waiting briefly for processing pipeline
* communication task waiting for transmit buffer space
* logger waiting for downstream consumer capacity

This is much closer to real embedded software than immediate-fail-only behavior.

# WNow compare **Step 2 vs Step 3**:

## Step 2

* immediate queue full

## Step 3

* bounded waiting for space

----

# 4️⃣ Queue Depth Monitoring and Saturation Visibility

## Goal of this step

Now we make the queue behavior **observable**, not just inferred.

So far you know:

* producer is faster
* consumer is slower
* timeout happens

Now we add:

> **queue depth monitoring**

So one can directly see:

* current queue occupancy
* when queue is nearly full
* when system is saturated

This is a very important industry practice:

> **Don’t just know that congestion exists — measure it.**

**What we will add**

We will use:

```c
osMessageQueueGetCount()
```

to measure how many items are currently in the queue.

Then we will print:

* queue depth after successful send
* queue depth after receive
* queue depth during timeout

## Replace `StartProducerTask()` with this

```c
void StartProducerTask(void *argument)
{
    uint32_t sensor_value = 100;
    osStatus_t status;
    char msg[180];
    uint32_t queue_count = 0;

    for(;;)
    {
        sensor_value += 10;

        /* Wait up to 200 ms for queue space */
        status = osMessageQueuePut(SensorQueueHandle, &sensor_value, 0, 200);
        queue_count = osMessageQueueGetCount(SensorQueueHandle);

        if (status == osOK)
        {
            g_produced_count++;

            snprintf(msg, sizeof(msg),
                     "ProducerTask: sent=%lu, produced=%lu, consumed=%lu, queue_count=%lu\r\n",
                     sensor_value,
                     g_produced_count,
                     g_consumed_count,
                     queue_count);

            uart_print(msg);
        }
        else if (status == osErrorTimeout)
        {
            snprintf(msg, sizeof(msg),
                     "ProducerTask: TIMEOUT, value=%lu, produced=%lu, consumed=%lu, queue_count=%lu\r\n",
                     sensor_value,
                     g_produced_count,
                     g_consumed_count,
                     queue_count);

            uart_print(msg);
        }
        else
        {
            snprintf(msg, sizeof(msg),
                     "ProducerTask: put failed, status=%d, value=%lu, queue_count=%lu\r\n",
                     (int)status,
                     sensor_value,
                     queue_count);

            uart_print(msg);
        }

        osDelay(100);
    }
}
```

## Replace `StartConsumerTask()` with this

```c
void StartConsumerTask(void *argument)
{
    uint32_t received_value = 0;
    osStatus_t status;
    char msg[180];
    uint32_t queue_count = 0;

    for(;;)
    {
        status = osMessageQueueGet(SensorQueueHandle, &received_value, NULL, osWaitForever);
        queue_count = osMessageQueueGetCount(SensorQueueHandle);

        if (status == osOK)
        {
            g_consumed_count++;

            snprintf(msg, sizeof(msg),
                     "ConsumerTask: got=%lu, produced=%lu, consumed=%lu, queue_count=%lu\r\n",
                     received_value,
                     g_produced_count,
                     g_consumed_count,
                     queue_count);

            uart_print(msg);

            osDelay(500);
        }
    }
}
```

***

## What you should observe now

You should start seeing queue occupancy clearly, for example:

```text
ProducerTask: sent=110, produced=1, consumed=0, queue_count=1
ConsumerTask: got=110, produced=1, consumed=1, queue_count=0
ProducerTask: sent=120, produced=2, consumed=1, queue_count=1
ProducerTask: sent=130, produced=3, consumed=1, queue_count=2
ProducerTask: sent=140, produced=4, consumed=1, queue_count=3
ProducerTask: sent=150, produced=5, consumed=1, queue_count=4
ProducerTask: sent=160, produced=6, consumed=1, queue_count=5
ProducerTask: TIMEOUT, value=170, produced=6, consumed=1, queue_count=5
ConsumerTask: got=120, produced=6, consumed=2, queue_count=4
```


## What this demonstrates

Now the behavior is much clearer:

* queue starts at low occupancy
* backlog increases as producer outpaces consumer
* queue reaches max capacity (`queue_count=5`)
* producer starts timing out
* consumer reduces occupancy gradually

This is an excellent observability improvement.


📌 `Backpressure is not just a logical condition — it is measurable through queue occupancy.`

📌 `In real systems, queue depth is a useful health indicator. It tells us whether the system is balanced or approaching saturation.`


## Strong industry analogy:

* sensor acquisition faster than processing
* logger generating messages faster than communication port can send
* CAN receive pipeline filling faster than downstream application handles frames

📌 `A queue is like a buffer in any real embedded pipeline — if downstream is slower, backlog grows.`

----

# 5️⃣ Improve System Balance by Making Consumer Faster

## Goal of this step

Now we do a **design improvement experiment**.

Instead of just observing saturation, we will change **one design parameter only**:

> make the **consumer faster**

This helps participants learn an important system design lesson:

> when a pipeline is congested, one solution is to improve downstream processing rate

## What we will change

Keep:

* Producer delay = **100 ms**
* Queue timeout = **200 ms**
* Queue length = **5**

Change only:

* Consumer processing delay from **500 ms** → **150 ms**

***

## Why this is the right next step

This gives a very clean comparison:

**Before**

* consumer too slow
* queue mostly full
* repeated timeouts

**After**

* consumer closer to producer speed
* queue occupancy should reduce
* timeouts should reduce significantly or disappear

## Replace `StartConsumerTask()` with this

```c
void StartConsumerTask(void *argument)
{
    uint32_t received_value = 0;
    osStatus_t status;
    char msg[180];
    uint32_t queue_count = 0;

    for(;;)
    {
        status = osMessageQueueGet(SensorQueueHandle, &received_value, NULL, osWaitForever);
        queue_count = osMessageQueueGetCount(SensorQueueHandle);

        if (status == osOK)
        {
            g_consumed_count++;

            snprintf(msg, sizeof(msg),
                     "ConsumerTask: got=%lu, produced=%lu, consumed=%lu, queue_count=%lu\r\n",
                     received_value,
                     g_produced_count,
                     g_consumed_count,
                     queue_count);

            uart_print(msg);

            osDelay(150);   // Faster consumer than previous step
        }
    }
}
```

## What you should expect now

You should start seeing behavior like:

```text
ProducerTask: sent=...
ConsumerTask: got=...
ProducerTask: sent=...
ConsumerTask: got=...
```

And importantly:

* `queue_count` should stay lower
* frequent `TIMEOUT` messages should reduce a lot
* produced and consumed counts should stay much closer

## What this step demonstrates

📌 `congestion can be reduced either by buffering more, slowing input, or improving downstream service rate`

In this step, we are testing: `What happens if we improve consumer throughput?`

📌 `When a queue is saturating, increasing queue size is not the only option. Often the better solution is to improve consumer processing rate.`

📌 `A queue depth problem is usually a system rate-matching problem.`

----

# 6️⃣ Larger Queue vs Faster Consumer — Buffering Trade-off

## Goal of this step

Keep the **consumer back to slower behavior**, but increase **queue capacity**.

Then observe:

* does the system *look* better?
* does timeout reduce?
* is the underlying rate mismatch really solved?

This is an important lesson:

> **A bigger queue can delay congestion, but it does not fix a permanently slow consumer.**

## In CubeMX / RTOS Queue configuration

Change queue length:

**Earlier**

```text
SensorQueue length = 5
```

**Now change to**

```text
SensorQueue length = 10
```

Then Generate Code.

## Change `ConsumerTask()` back to slow version; Replace only `StartConsumerTask()` with:

```c
void StartConsumerTask(void *argument)
{
    uint32_t received_value = 0;
    osStatus_t status;
    char msg[180];
    uint32_t queue_count = 0;

    for(;;)
    {
        status = osMessageQueueGet(SensorQueueHandle, &received_value, NULL, osWaitForever);
        queue_count = osMessageQueueGetCount(SensorQueueHandle);

        if (status == osOK)
        {
            g_consumed_count++;

            snprintf(msg, sizeof(msg),
                     "ConsumerTask: got=%lu, produced=%lu, consumed=%lu, queue_count=%lu\r\n",
                     received_value,
                     g_produced_count,
                     g_consumed_count,
                     queue_count);

            uart_print(msg);

            osDelay(500);   // Slow consumer again
        }
    }
}
```

## Keep `ProducerTask()` exactly as in Step 4 / Step 5

That means:

* producer delay = **100 ms**
* put timeout = **200 ms**
* queue count printing enabled

Keep it unchanged.

## What you should expect now

You will likely see:

* queue count rising from 1 → 2 → 3 ... up to 10
* timeout starts **later than before**
* once queue fills to 10, timeout comes back
* produced and consumed gap becomes larger

Typical behavior may look like:

```text
ProducerTask: sent=110, produced=1, consumed=0, queue_count=1
ProducerTask: sent=120, produced=2, consumed=0, queue_count=2
...
ProducerTask: sent=200, produced=10, consumed=1, queue_count=9
ProducerTask: sent=210, produced=11, consumed=1, queue_count=10
ProducerTask: TIMEOUT, value=220, produced=11, consumed=1, queue_count=10
```

## What this step demonstrates

This is the key lesson:

**Larger queue helps by**:

* absorbing bursts
* delaying congestion
* smoothing temporary mismatches

**But larger queue does **not** solve**:

* permanent producer > consumer rate mismatch

📌 `A larger queue is useful for burst absorption, not as a substitute for proper rate matching.`

📌 `If producer is permanently faster than consumer, even a big queue will eventually fill.`

## Which is better?

1. **Make consumer faster**
2. **Increase queue size**

**Correct answer**:

It depends on system requirement, but generally:

* use **larger queue** for burst tolerance
* use **faster consumer / better architecture** for sustained load mismatch

----