# Exercise 1

# 1️⃣ Superloop Baseline: Heartbeat + Logger

## What this demo should show
This is a small embedded application where:

* LED blinks every 500 ms
* UART prints a heartbeat message every cycle

## Add this helper function above main()

````c
void uart_print(char *msg)
{
    HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), HAL_MAX_DELAY);
}
````

## Replace the while (1) loop with this

````c
while (1)
{
    HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
    uart_print("Superloop heartbeat\r\n");
    HAL_Delay(500);
}
````

## Full expected logic
This code does only 3 things:

1. Toggle LED
2. Print message on UART
3. Delay 500 ms
4. Then repeat.

----

# 2️⃣ Superloop with Button Polling

## Goal of this step
Now we add a second responsibility to the same superloop:

* LED heartbeat
* UART log
* **Button polling**

At this stage, the code still works, but now the single-loop structure starts coupling behaviors together.

## Add this variable before the while(1) loop

````c
GPIO_PinState last_button_state = HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13);
````

## Replace the while(1) loop with this

````c
while (1)
{
    GPIO_PinState current_button_state = HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13);

    HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
    uart_print("Superloop heartbeat\r\n");

    if ((current_button_state == GPIO_PIN_SET) && (last_button_state == GPIO_PIN_RESET))
    {
        uart_print("Button press detected in superloop\r\n");
    }

    last_button_state = current_button_state;

    HAL_Delay(500);
}
````

## What this code is doing
Now your single superloop is doing three responsibilities:

1. Read button state
2. Blink LED
3. Print UART messages
4. Delay 500 ms
5. Repeat

# Important observation
If you press and release the button quickly, the press may not get detected. Because the loop checks the button only once per cycle, and then spends 500 ms inside HAL_Delay().

📌 `The application is still simple, but responsiveness is already getting tied to one big loop.`

📌 `Nothing is wrong with this code yet. But now you can see that LED timing, UART logging, and button response are all tied to the same loop timing.`

----

# 3️⃣ Expose the Limitation of Superloop

## Goal of this step
We will now intentionally make the superloop worse so participants can clearly see the problem:

* LED heartbeat
* Button polling
* UART logging
* Extra processing delay

## Add this helper function above main()

````c
void simulate_processing_load(void)
{
    for (volatile uint32_t i = 0; i < 800000; i++)
    {
        // Intentional empty loop to simulate CPU workload
    }
}
````

## Replace the while(1) loop with this
````c
while (1)
{
    GPIO_PinState current_button_state = HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13);

    HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
    uart_print("Superloop heartbeat\r\n");

    if ((current_button_state == GPIO_PIN_SET) && (last_button_state == GPIO_PIN_RESET))
    {
        uart_print("Button press detected in superloop\r\n");
    }

    last_button_state = current_button_state;

    uart_print("Simulating extra processing...\r\n");
    simulate_processing_load();

    HAL_Delay(500);
}

````

## What this code is doing now
Now the single loop is handling:

1. Button polling
2. LED heartbeat
3. UART logging
4. Artificial processing load
5. Delay
6. Repeat

So one loop is now carrying multiple responsibilities + workload.

## Obervation

You should notice:

* button response feels worse
* quick press may be missed
* heartbeat timing is no longer “cleanly tied” to only LED behavior
* everything is coupled to one large loop cycle

📌 `The problem is not that the code is incorrect. The problem is that timing, responsiveness, and behavior of all activities are now coupled to one loop.`

Why this step matters

* every new activity makes loop timing worse
* polling quality depends on loop execution time
* communication and processing affect event handling

This is the practical reason for moving to task-based design.

----

# 4️⃣ Convert Superloop to RTOS Tasks

## Goal of this step
Take the same responsibilities and separate them into independent RTOS tasks:

* Heartbeat Task → LED blink
* Button Task → button polling
* Logger Task → UART status print

📌 This is the point where `same application behavior, but with cleaner structure and independent timing`

## What to configure first in CubeMX
Do this first before code.

* Enable FreeRTOS
    * Middleware → FreeRTOS
    * Interface: keep default
    * CMSIS version: CMSIS_V2

* Create 3 tasks in FreeRTOS configuration:
    
    Task 1
    * Name: HeartbeatTask
    * Priority: Normal
    * Stack size: 128 words

    Task 2
    * Name: ButtonTask
    * Priority: Normal
    * Stack size: 128 words

    Task 3
    * Name: LoggerTask
    * Priority: Low
    * Stack size: 256 words

* Keep these peripherals enabled

    * PA5 → LED output
    * PC13 → button input
    * USART2 → UART at 115200

* Generate code

## Add these global variables and helper function in USER CODE section

````c
volatile uint32_t g_button_press_count = 0;
volatile uint8_t g_button_event_flag = 0;

void uart_print(char *msg)
{
    HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), HAL_MAX_DELAY);
}
````

## Replace the body of HeartbeatTask function with this

Use the task function generated by CubeMX for HeartbeatTask
(name may look like StartHeartbeatTask(void *argument))

````c
void StartHeartbeatTask(void *argument)
{
    for(;;)
    {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
        osDelay(500);
    }
}
````

## Replace the body of ButtonTask function with this

````c
void StartButtonTask(void *argument)
{
    GPIO_PinState last_button_state = HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13);

    for(;;)
    {
        GPIO_PinState current_button_state = HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13);

        if ((current_button_state == GPIO_PIN_SET) && (last_button_state == GPIO_PIN_RESET))
        {
            g_button_press_count++;
            g_button_event_flag = 1;
        }

        last_button_state = current_button_state;

        osDelay(50);
    }
}
````

## Replace the body of LoggerTask function with this

````c
void StartLoggerTask(void *argument)
{
    char msg[100];

    for(;;)
    {
        if (g_button_event_flag)
        {
            snprintf(msg, sizeof(msg), "Button press detected. Count = %lu\r\n", g_button_press_count);
            uart_print(msg);
            g_button_event_flag = 0;
        }

        snprintf(msg, sizeof(msg), "RTOS logger alive. Button count = %lu\r\n", g_button_press_count);
        uart_print(msg);

        osDelay(1000);
    }
}
````

## Observe now

On hardware : LED blinks every 500 ms

On serial terminal: You should see something like

````
RTOS logger alive. Button count = 0
RTOS logger alive. Button count = 0
Button press detected. Count = 1
RTOS logger alive. Button count = 1
RTOS logger alive. Button count = 1
````

## What this step demonstrates
Now the same application is expressed as:

* one task for status heartbeat
* one task for input monitoring
* one task for communication/logging

This is much closer to real embedded system organization.

📌 `The CPU is still one, but the application is now expressed as separate responsibilities with independent timing.`

📌 `This is the real value of RTOS at this stage — structure, readability, and controlled timing.`