# RTOS Embedded Systems Training (STM32 + FreeRTOS)

This repository provides a structured, hands-on approach to learning Real-Time Operating Systems (RTOS) using STM32 and FreeRTOS.

The content progresses from basic superloop-based design to a modular, task-driven embedded system architecture.

---

## Learning Progression

1. Superloop vs RTOS transition  
2. Task creation and execution  
3. Scheduler behavior and task priorities  
4. RTOS debugging and task visibility  
5. Queue-based inter-task communication  
6. ISR to task synchronization using semaphore  
7. Multi-stage data pipeline design  
8. Shared resource protection using mutex  

---

## Repository Structure

| Folder | Description |
|--------|------------|
| [01_superloop_vs_rtos](./01_superloop_vs_rtos/) | Transition from superloop to RTOS |
| [02_task_creation](./02_task_creation/) | Creating and running multiple tasks |
| [03_scheduler_behavior](./03_scheduler_behavior/) | Task priorities and starvation |
| [04_rtos_debugging](./04_rtos_debugging/) | Debugging RTOS systems |
| [05_queue_communication](./05_queue_communication/) | Inter-task communication using queues |
| [06_semaphore_isr_signaling](./06_semaphore_isr_signaling/) | ISR to task synchronization |
| [07_data_pipeline](./07_data_pipeline/) | Multi-stage pipeline design |
| [08_mutex_shared_uart](./08_mutex_shared_uart/) | Shared resource protection using mutex |

---

## Key Concepts Covered

- RTOS fundamentals
- Task creation and scheduling
- Preemptive priority-based execution
- Inter-task communication using queues
- Producer–consumer architecture
- ISR-to-task signaling using semaphores
- Shared resource protection using mutex
- Embedded system pipeline design
- Runtime debugging and visibility

---

## Hardware Platform

- STM32F446RE Nucleo Board
- Onboard LED (LD2)
- User Button (PC13)
- ADC input (PA0 – potentiometer)

---

## Design Philosophy

- Focus on **industry-relevant embedded system design**
- Emphasis on **architecture and modularity**
- Step-by-step evolution from simple to complex systems
- Clear separation of responsibilities across tasks
- Minimal but effective demonstration code

---

## How to Use

1. Follow folders sequentially from `01` to `08`
2. Build and run each project independently
3. Observe system behavior using UART logs or debugger
4. Analyze progression from superloop → RTOS → structured pipeline

---

## Target Audience

- Engineering Faculty
- Final-year undergraduate students
- Embedded systems learners transitioning to RTOS

---

## Future Extensions

- ADC filtering and signal processing
- CAN communication tasks
- Sensor fusion pipelines
- Automotive ECU simulation examples

---
