---
layout: page
title: Tetris on STM32 (LED Matrix Game Engine)
permalink: /tetris.html
---

## Summary

- **Snapshot** — Tetris-like gameplay implemented on a 64x64 RGB LED matrix, driven by STM32F0.  
- **Core Components** — GPIO-driven row/column selection, timer-based refresh, SPI/DMA for peripheral displays, and EXTI interrupts for keypad control.  
- **Implementation Choice** — **C (STM32 HAL-free, register-level)** with modular sections: pin definitions, matrix drivers, game logic, interrupts, and display management.  

<!-- ![Summary]({{ "/doc/tetris-summary.png" | relative_url }}) -->

---

## Pin Definition

- **Row select (A–D)** mapped to **GPIOA9–12**.  
- **Color channels (R/G/B)** mapped to **GPIOC0–5** for upper/lower panels.  
- **Control signals:** `CLOCK (PC6)`, `LATCH (PC7)`, and `OE (PC8)` for LED driver shift registers.  
- Mapping constants (`#define`) give readable labels (e.g., `ROW_A`, `R1`) for hardware control.  

This ensures hardware portability and simplifies low-level GPIO manipulation.

---

## Function Prototypes

The header declares all core functions up front:
- **Matrix operations:** `set_row`, `color_lowpanel`, `pulse_clock`, `latch_data`.  
- **Game logic:** `drawBlock`, `clearBlock`, `start_Game`, `game_over`.  
- **Interrupts:** `EXTI0_1_IRQHandler`, `EXTI2_3_IRQHandler`.  
- **Peripherals:** `init_spi1`, `spi1_setup_dma`, `setup_tim14`.  

This separation makes the file navigable and supports modular development.

---

## Matrix Functions

- **Row scanning:** `set_row()` energizes one row at a time.  
- **Color setting:** `color_lowpanel()` and `color_uppanel()` assign RGB values per panel.  
- **Clock & latch:** `pulse_clock()` shifts in 32 bits per row, `latch_data()` stores the values.  
- **Frame refresh:** `lightup_wholeMatrix()` iterates over the 64x64 grid (`arr_map`) and updates LED states.  

Together, these provide the foundation for visualizing blocks, cleared lines, and animations.

---

## Interrupts

- **EXTI0_1_IRQHandler:** Handles button events for downward movement and left shifts.  
- **EXTI2_3_IRQHandler:** Handles button events for right shifts and reset.  
- **Timer interrupts (TIM7, TIM14):** Periodically refresh the LED matrix and score display.  

These interrupts decouple **user input** and **game progression** from the main loop, enabling responsive controls and consistent display refresh.

---

## Peripheral Setup

- **GPIO:** `enable_ports()` configures matrix pins, keypad inputs, and indicator LEDs.  
- **Timers:**  
  - `TIM7` → refresh scanning of LED display.  
  - `TIM14` → update score every 100 ms.  
  - `TIM3` → generate periodic signals.  
- **SPI1 & SPI2:** drive external displays (7-seg and OLED) using DMA for efficient transfers.  
- **DMA:** moves frame/character buffers directly to SPI peripherals.  

This ensures hardware accelerates repetitive tasks (refresh, data transfer) while CPU focuses on game logic.

---

## Game Logic

- **Block spawning:** `start_Game()` selects a random tetromino from `TETROMINOES[][][]`.  
- **Block placement:** `drawBlock()` writes the block into `arr_map`, `clearBlock()` erases it.  
- **Collision detection:** `check_collision()` ensures pieces stack properly.  
- **Line clearing:** `should_shift()` detects filled rows, shifts the array, and increments `score_game`.  
- **Win/Lose states:** `show_WINNING()` fills the matrix with green; `show_DUMA()` displays a “GAME OVER” pattern.  

The game loop in `main()` ties these together, continuously checking state and updating display.

---

## Semaphore

**Semaphores** are used to manage shared resources:  
- **Shared resources:** `arr_map` (game grid) and `disp` (score buffer).  
- **Race condition risk:** both **interrupts** (button presses) and **main loop** update `arr_map`.  
- **Semaphore role:**  
  - A binary semaphore could guard `arr_map` during updates, ensuring atomic writes when a block moves down (main loop) vs. sideways (interrupt).  
  - Counting semaphores could regulate refresh vs. input frequency (e.g., limit moves per frame).  

Adding semaphores makes the game robust under concurrent events!

---

## Result

- **Working Demo:** STM32F0 running Tetris on a 64x64 LED matrix, responsive to keypad input, with score displayed on 7-seg/OLED.  
- **Video:** [Watch Demo Here](https://www.youtube.com/watch?v=aQfffqlmvk8)

---
