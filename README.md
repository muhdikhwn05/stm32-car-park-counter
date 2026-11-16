# STM32 Car Park System 🚗

An embedded car park counter system built using bare-metal register manipulation on the **STM32L432KC Nucleo-32** board. This project tracks vehicle entries and exits, displays the current availability on a 7-segment display, and triggers a "FULL" animation sequence when capacity is reached.

**Course:** SKEE3223-02 Embedded Systems (Group 5) - Universiti Teknologi Malaysia (UTM)  
**Team Members:** Muhammad Ikhwan, Samuel Calvin, Nabil Nazmi, Muhamamad Izzuddin, Muhammad Danish Zikrullah  

## 🎯 Features
*   **Bare-Metal GPIO Configuration:** Bypasses HAL for GPIO initialization, directly manipulating `MODER`, `OTYPER`, and `PUPDR` registers.
*   **Capacity Tracking (0-10):** Automatically increments on entry and decrements on exit.
*   **Hardware Debouncing Logic:** Implements polling delays to ensure accurate button presses.
*   **Custom FULL Animation:** When the counter hits 10, the green LED turns off, and the 7-segment display cycles through the letters `F -> U -> L -> L`.

## 🔌 Hardware & Pin Configuration

| Component | Function | STM32 Pin (MCU) | Nucleo Label | Configuration Mode |
| :--- | :--- | :--- | :--- | :--- |
| **Button 1** | Entry Button | PA0 | A0 | Input (Pull-Up) |
| **Button 2** | Exit Button | PA12 | D2 | Input (Pull-Up) |
| **7-Segment** | Segment A | PA1 | A1 | Output (Push-Pull) |
| **7-Segment** | Segment B | PA3 | A2 | Output (Push-Pull) |
| **7-Segment** | Segment C | PA4 | A3 | Output (Push-Pull) |
| **7-Segment** | Segment D | PA5 | A4 | Output (Push-Pull) |
| **7-Segment** | Segment E | PA8 | D9 | Output (Push-Pull) |
| **7-Segment** | Segment F | PA7 | A6 | Output (Push-Pull) |
| **7-Segment** | Segment G | PA2 | A7 | Output (Push-Pull) |
| **LED** | Vacancy Indicator | PB6 | D5 | Output (Push-Pull) |

## 🛠️ System Logic Flow
1. The system initializes with the Green LED ON (indicating vacancy) and the counter at 0.
2. Pressing the **Entry Button (PA0)** increases the counter (Max 10).
3. Pressing the **Exit Button (PA12)** decreases the counter (Min 0).
4. If the counter reaches **10**, the Green LED turns off, and the 7-segment display loops an animation spelling "F-U-L-L".

## 🚀 How to Run
1. Open the project in **STM32CubeIDE**.
2. Build the project and flash it to the STM32L432KC Nucleo-32 board.
3. Wire the components on a breadboard according to the pinout table above.

## Schematic
![Wokwi Schematic](https://github.com/user-attachments/assets/7b87e0f1-703c-4955-8657-0aeba67b10ad)

## Hardware Setup
![Breadboard Setup](https://github.com/user-attachments/assets/be7b3d76-91e1-447d-9322-e0ec46c66f76)


## Coding
#include "main.h"

// Macro definitions for delay and 7-segment pins
#define delay_ms(x) HAL_Delay(x)
#define SEG_A (1<<1)  // Pin A1
#define SEG_B (1<<3)  // Pin A2
#define SEG_C (1<<4)  // Pin A3
#define SEG_D (1<<5)  // Pin A4
#define SEG_E (1<<8)  // Pin D9
#define SEG_F (1<<7)  // Pin A6
#define SEG_G (1<<2)  // Pin A7

// Function Prototypes
static void MX_GPIO_Init(void);
void GPIO_Init_Manual(void);

int main(void)
{
    // Initialize the HAL Library and System Clock
    HAL_Init();
    SystemClock_Config();
    
    // Initialize GPIOs using bare-metal registers
    GPIO_Init_Manual();

    int i = 0; // Counter for cars

    while (1)
    {
        // --- Button 1 Entry (Pin A0 / PA0) ---
        if ((GPIOA->IDR & (1<<0)) == 0)
        {
            delay_ms(50); // Debounce delay
            if ((GPIOA->IDR & (1<<0)) == 0)
            {
                if (i < 10) i++;
                while ((GPIOA->IDR & (1<<0)) == 0); // Wait for release
            }
        }

        // --- Button 2 Exit (Pin D2 / PA12) ---
        if ((GPIOA->IDR & (1<<12)) == 0)
        {
            delay_ms(50); // Debounce delay
            if ((GPIOA->IDR & (1<<12)) == 0)
            {
                if (i > 0) i--; // Only remove if not empty
                while ((GPIOA->IDR & (1<<12)) == 0); // Wait for release
            }
        }

        // --- Display Logic ---
        uint32_t mask = 0xFFFF;

        if (i == 0)
            mask = ~(SEG_A | SEG_F | SEG_D | SEG_E | SEG_G); // Show 0
        else if (i == 1)
            mask = ~(SEG_B | SEG_C); // Show 1
        else if (i == 2)
            mask = ~(SEG_A | SEG_B | SEG_D | SEG_E | SEG_G); // Show 2
        else if (i == 3)
            mask = ~(SEG_A | SEG_B | SEG_C | SEG_D | SEG_G); // Show 3
        else if (i == 4)
            mask = ~(SEG_B | SEG_C | SEG_F | SEG_G); // Show 4
        else if (i == 5)
            mask = ~(SEG_A | SEG_C | SEG_D | SEG_F | SEG_G); // Show 5
        else if (i == 6)
            mask = ~(SEG_A | SEG_C | SEG_D | SEG_E | SEG_F | SEG_G); // Show 6
        else if (i == 7)
            mask = ~(SEG_A | SEG_B | SEG_C); // Show 7
        else if (i == 8)
            mask = ~(SEG_A | SEG_B | SEG_C | SEG_D | SEG_E | SEG_F | SEG_G); // Show 8
        else if (i == 9)
            mask = ~(SEG_A | SEG_B | SEG_C | SEG_D | SEG_F | SEG_G); // Show 9

        // If not full, display number and turn ON green LED
        if (i != 10) {
            GPIOA->ODR = mask;
            GPIOB->ODR |= (1<<6); // Green LED ON
        }
        else if (i == 10)
        {
            GPIOB->ODR &= ~(1<<6); // Green LED OFF
            
            // FRAME 1: F
            GPIOA->ODR = 0xFFFF & ~(SEG_A | SEG_F | SEG_E | SEG_G);
            delay_ms(500);
            if ((GPIOA->IDR & (1<<12)) == 0) {
                i = 9; while ((GPIOA->IDR & (1<<12)) == 0); continue;
            }

            // FRAME 2: U
            GPIOA->ODR = 0xFFFF & ~(SEG_B | SEG_C | SEG_D | SEG_E | SEG_F);
            delay_ms(500);
            if ((GPIOA->IDR & (1<<12)) == 0) {
                i = 9; while ((GPIOA->IDR & (1<<12)) == 0); continue;
            }

            // FRAME 3: L
            GPIOA->ODR = 0xFFFF & ~(SEG_D | SEG_E | SEG_F);
            delay_ms(500);
            if ((GPIOA->IDR & (1<<12)) == 0) {
                i = 9; while ((GPIOA->IDR & (1<<12)) == 0); continue;
            }

            // FRAME 4: L
            GPIOA->ODR = 0xFFFF & ~(SEG_D | SEG_E | SEG_F);
            delay_ms(500);
        }
    }
}

void GPIO_Init_Manual(void)
{
    // Enable Peripheral Clocks: Bit 0 = GPIOA, Bit 1 = GPIOB
    RCC->AHB2ENR |= (1 << 0) | (1 << 1);

    // --- Configure 7-Segment Pins (Port A) as Output ---
    // Clear bits (masking with 3 shifted to position clears both bits for that pin)
    GPIOA->MODER &= ~( (3U<<(1*2)) | (3U<<(2*2)) | (3U<<(3*2)) | (3U<<(4*2)) | (3U<<(5*2)) | (3U<<(7*2)) | (3U<<(8*2)) );
    // Set '01' for Output
    GPIOA->MODER |=  ( (1U<<(1*2)) | (1U<<(2*2)) | (1U<<(3*2)) | (1U<<(4*2)) | (1U<<(5*2)) | (1U<<(7*2)) | (1U<<(8*2)) );
    // Configure OTYPER: 0 = Push-Pull
    GPIOA->OTYPER &= ~(SEG_A | SEG_B | SEG_C | SEG_D | SEG_E | SEG_F | SEG_G);

    // --- Configure PB6 (Green LED) as Output ---
    GPIOB->MODER &= ~(3U << (6*2)); // Clear mode bits
    GPIOB->MODER |=  (1U << (6*2)); // Set to Output (01)
    GPIOB->OTYPER &= ~(1U << 6);    // Push-Pull
    GPIOB->OSPEEDR &= ~(3U << (6*2)); // Optional: Set speed to low

    // --- Configure INPUT Pins (PA0 and PA12) ---
    // MODER: 00 = Input mode
    GPIOA->MODER &= ~( (3U<<(0*2)) | (3U<<(12*2)) );
    GPIOA->PUPDR &= ~( (3U<<(0*2)) | (3U<<(12*2)) );
    // Set pull-up resistor to both switch PA0 and PA12
    GPIOA->PUPDR |=  ( (1U<<(0*2)) | (1U<<(12*2)) );
}
