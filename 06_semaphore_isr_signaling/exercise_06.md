# Exercise 6

# 1️⃣ Binary Semaphore for ISR-to-Task Signaling**

## Goal of this step

Show this architecture clearly:

```text
Button Press → ISR → Give Semaphore → Waiting Task Unblocks → Task Processes Event
```

This is a **very important embedded systems design pattern**.


## What this step will demonstrate

* Button press generates an interrupt
* ISR does **not** print or process heavily
* ISR only signals using a **binary semaphore**
* Processing task wakes up and logs the event

## CubeMX / CubeIDE configuration

* UART
    * Enable:
    * **USART2**
    * Asynchronous
    * Baud = **115200**

We will continue using `uart_print()` for reliable output.


* Button interrupt
    * Use onboard button: **PC13 / B1**
    * Configure as: **GPIO\_EXTI13**
    * Trigger: **Falling edge**  
        *(typical for Nucleo user button)*
        > If your board behavior is opposite, we can adjust later.


* LED output: Keep onboard LED:
    * **PA5 / LD2**
    * Output push-pull

* FreeRTOS: Enable FreeRTOS and create:
    * Task
        * **Name:** `EventTask`
        * **Priority:** Normal
        * **Stack size:** 256 words
    * Add a binary semaphore
        * **Name:** `ButtonSem`

Then generate code.

## Add these global variables

```c
volatile uint32_t g_button_isr_count = 0;
volatile uint32_t g_button_task_count = 0;
```


## Keep this UART helper

```c
void uart_print(char *msg)
{
    HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), HAL_MAX_DELAY);
}
```

If not already present, keep these includes at top:

```c
#include <stdio.h>
#include <string.h>
```


## Replace `StartEventTask()` (or your generated task function) with this

Use your actual generated task function name. Example:

```c
void StartEventTask(void *argument)
{
    char msg[120];

    for(;;)
    {
        osSemaphoreAcquire(ButtonSemHandle, osWaitForever);

        g_button_task_count++;

        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);

        snprintf(msg, sizeof(msg),
                 "EventTask: button event processed, isr_count=%lu, task_count=%lu\r\n",
                 g_button_isr_count,
                 g_button_task_count);

        uart_print(msg);
    }
}
```

***

## Add ISR callback

Add this function:

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == GPIO_PIN_13)
    {
        g_button_isr_count++;
        osSemaphoreRelease(ButtonSemHandle);
    }
}
```

***

## What you should observe

When you press the button:

* LED toggles
* UART prints one message per processed event

Typical output:

```text
EventTask: button event processed, isr_count=1, task_count=1
EventTask: button event processed, isr_count=2, task_count=2
EventTask: button event processed, isr_count=3, task_count=3
```

***

# What this step demonstrates

This is the key lesson:

* ISR does **minimal work**
* Task does the visible processing
* Semaphore is the synchronization bridge between interrupt context and task context

This is exactly the design pattern you want faculty to learn.


📌 `ISR should be short and deterministic. It should signal work, not perform heavy application logic.`

📌 `The task is the correct place for logging, processing, and application behavior.`