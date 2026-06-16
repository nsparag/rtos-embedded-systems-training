# Exercise 2

# 1️⃣ Create Two Independent RTOS Tasks

## Goal of this step
This step is not yet about priorities. It is only about making participants clearly see:

`one application can be split into multiple independently scheduled tasks`

To keep it clean and visible, we will use:

* Task 1: LED heartbeat task
* Task 2: UART logger task

This is the simplest solid demo for task creation + independent timing.

## CubeMX / CubeIDE configuration

* GPIO
    * PA5 → Output (LD2 onboard LED)

* UART
    * USART2
    * Asynchronous
    * Baud rate = 115200

* FreeRTOS: Enable FreeRTOS and create 2 tasks:
    * Task 1
        * Name: BlinkTask
        * Priority: Normal
        * Stack size: 128 words

    * Task 2
        * Name: LoggerTask
        * Priority: Low
        * Stack size: 256 words

Then generate code.

## Add this helper function in USER CODE section
```C
void uart_print(char *msg)
{
    HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), HAL_MAX_DELAY);
}
```

## Put this code inside the BlinkTask function

Use the exact generated function name from CubeMX (for example: StartBlinkTask(void *argument))

```C
void StartBlinkTask(void *argument)
{
    for(;;)
    {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
        uart_print("BlinkTask: LED toggled\r\n");
        osDelay(1000);
    }
}
```

## Put this code inside the LoggerTask function
```c
void StartLoggerTask(void *argument)
{
    uint32_t log_count = 0;
    char msg[80];

    for(;;)
    {
        log_count++;
        snprintf(msg, sizeof(msg), "LoggerTask: system alive, count = %lu\r\n", log_count);
        uart_print(msg);
        osDelay(300);
    }
}

```

## What you should observe

On hardware: LED blinks every 1000 ms

On terminal: You should continuously see output like:

```text
LoggerTask: system alive, count = 1
LoggerTask: system alive, count = 2
LoggerTask: system alive, count = 3
BlinkTask: LED toggled
LoggerTask: system alive, count = 4
LoggerTask: system alive, count = 5
LoggerTask: system alive, count = 6
BlinkTask: LED toggled
```

## What this step demonstrates

This clearly shows:
* two separate tasks exist
* both have their own timing
* one task handles hardware heartbeat
* one task handles periodic logging
* the scheduler is switching between them

📌 `There is still only one CPU, but the application is now expressed as separate responsibilities with separate timing.`

📌 `This is the first advantage of RTOS — clean separation, not API complexity`

----

# 2️⃣ Three Independent Periodic Tasks

## Goal of this step
Now we make task independence more visible by having three separate responsibilities, each with its own rate:

* BlinkTask → LED heartbeat every 1000 ms
* LoggerTask → fast log every 300 ms
* StatusTask → slow status message every 2000 ms

This makes the scheduler behavior easier to observe without going into priorities yet.

## Add one more task in FreeRTOS

* Task 3
    * Name: StatusTask
    * Priority: Low
    * Stack size: 256 words

Then Generate Code.

## Replace the body of BlinkTask with this

````c
void StartBlinkTask(void *argument)
{
    uint32_t blink_count = 0;
    char msg[80];

    for(;;)
    {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
        blink_count++;

        snprintf(msg, sizeof(msg), "BlinkTask   : LED toggled, count = %lu\r\n", blink_count);
        uart_print(msg);

        osDelay(1000);
    }
}
````

## Replace the body of LoggerTask with this

````c
void StartLoggerTask(void *argument)
{
    uint32_t log_count = 0;
    char msg[80];

    for(;;)
    {
        log_count++;

        snprintf(msg, sizeof(msg), "LoggerTask  : fast periodic log, count = %lu\r\n", log_count);
        uart_print(msg);

        osDelay(300);
    }
}
````

## Put this code inside the new StatusTask function

````c
void StartStatusTask(void *argument)
{
    uint32_t status_count = 0;
    char msg[80];

    for(;;)
    {
        status_count++;

        snprintf(msg, sizeof(msg), "StatusTask  : slow health status, count = %lu\r\n", status_count);
        uart_print(msg);

        osDelay(2000);
    }
}
````

## observe

On hardware: LED toggles every 1000 ms

On terminal: You should see mixed output like this:
````
LoggerTask  : fast periodic log, count = 1
LoggerTask  : fast periodic log, count = 2
LoggerTask  : fast periodic log, count = 3
BlinkTask   : LED toggled, count = 1
LoggerTask  : fast periodic log, count = 4
LoggerTask  : fast periodic log, count = 5
LoggerTask  : fast periodic log, count = 6
StatusTask  : slow health status, count = 1
BlinkTask   : LED toggled, count = 2
````

## What this step demonstrates
This shows clearly:

* multiple tasks can coexist cleanly
* each task has its own execution rate
* periodic responsibilities are easier to express in RTOS
* the application is becoming more like a real embedded software structure

📌 `Notice that each behavior now has its own natural timing. We are no longer trying to manually juggle everything inside one loop.`

📌 `This is how embedded software starts becoming modular and maintainable.`

----