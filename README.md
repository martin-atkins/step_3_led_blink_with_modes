# step_2_led_blink_non-blocking
Building from step 1 with non-blocking

## Using these versions:
* `STM32CubeIDE` : `1.18.1`
* `STM32CubeMX` : `6.14.1`

## The right ways to blink without `HAL_Delay()`
* Option 1 — Timer interrupt
  * hardware timer generates periodic interrupts
  * LED toggles in ISR
  * CPU stays free
  * deterministic timing
* Option 2 — SysTick / timebase + state machine
  * poll `HAL_GetTick()`
  * non-blocking, but...
    * still tied to SysTick
 
**I’ll do Option 1: TIM2 interrupt**

### Part A — Configure TIM2 in CubeMX
* Open my `.ioc` file
  * **Enable TIM2**
    * Pinout & Configuration
    * Timers → TIM2
    * Set only
      * **Clock Source**: `Internal Clock`
    * Leave:
      * Slave Mode: `Disabled`
      * Trigger Source: `Disabled`
      * Channels 1–4: `Disabled`
      * Combined Channels: `Disabled`
      * One Pulse Mode: `unchecked`
  * **Configure TIM2 parameters**
    * Click **TIM2 → Configuration → Parameter Settings**
    * Set:
      * **Prescaler (PSC)**: `16000 - 1`
      * **Counter Period (ARR)**: `1000 - 1`
      * **Auto-reload preload**: `Enabled`
      * **Counter mode**: `Up`
    * Why?
      * `APB1` timer clock ≈ 16 MHz
      * Prescaler → 1 kHz
      * Period → 1 second
  * **Enable TIM2 interrupt**
    * Still in **TIM2 → Configuration:**
      * Go to **NVIC Settings**
      * Enable **TIM2 global interrupt**
  * **Save** to generate code
    * I should now have...
      * `MX_TIM2_Init()` generated
      * `htim2` declared
      * NVIC configured
### Coding updates
* In `main.c`:
  * **Start timer with interrupts**
    * inside `main()`:
      ```
      MX_TIM2_Init();
      HAL_TIM_Base_Start_IT(&htim2);
      ```
  * **Callback implementation**
    * add **outside** `main()`:
      ```
      void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
      {
          if (htim->Instance == TIM2)
          {
              HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin);
          }
      }
      ```
## UPDATE : replace the LED toggle with a software state machine driven by the timer
### Design first
* **Rule 1** — ISRs should be dumb
  * Interrupts should:
    * Tick time
    * Set flags
    * Never contain behavior logic
* **Rule 2** — Behavior lives in the main loop
  * The main loop:
    * Runs a state machine
    * Reacts to time events
    * Is testable and scalable

### Target behavior
* We’ll implement 3 LED modes:
  * Mode	Pattern
    * `MODE_OFF`	LED off
    * `MODE_SLOW`	Blink 1 Hz
    * `MODE_FAST`	Blink 5 Hz

### Step 1 — Reconfigure TIM2 (10 ms tick)
* In CubeMX, adjust TIM2:
  * **Prescaler**: `16000 - 1`
  * **Period**: `10 - 1`

Then regenerate code.
### Step 2 — Global timing variables
* Add near the top of `main.c`:
  ```
  volatile uint32_t system_tick_ms = 0;
  ```
### Step 3 — Minimal ISR (tick only)
* Replace the callback with this:
  ```
  void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
  {
      if (htim->Instance == TIM2)
      {
          system_tick_ms++;
      }
  }
  ```
### Step 4 — Define the LED state machine
* Add above `main()`:
  ```
  typedef enum
  {
      LED_MODE_OFF = 0,
      LED_MODE_SLOW,
      LED_MODE_FAST
  } led_mode_t;
  
  typedef struct
  {
      led_mode_t mode;
      uint32_t   last_toggle_ms;
      uint32_t   interval_ms;
      uint8_t    led_on;
  } led_ctrl_t;
  ```
### Step 5 — Initialize the controller
* Inside `main()` **after GPIO init**:
  ```
  led_ctrl_t led =
  {
      .mode = LED_MODE_SLOW,
      .last_toggle_ms = 0,
      .interval_ms = 500,
      .led_on = 0
  };
  ```
### Step 6 — State machine update function
* Add this below `main()`:
  ```
  void led_update(led_ctrl_t *ctrl, uint32_t now_ms)
  {
      switch (ctrl->mode)
      {
          case LED_MODE_OFF:
              HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_RESET);
              ctrl->led_on = 0;
              break;
            case LED_MODE_SLOW:
              ctrl->interval_ms = 500;
              break;
            case LED_MODE_FAST:
              ctrl->interval_ms = 100;
              break;
      }
  
      if (ctrl->mode != LED_MODE_OFF)
      {
          if ((now_ms - ctrl->last_toggle_ms) >= ctrl->interval_ms)
          {
              ctrl->last_toggle_ms = now_ms;
              ctrl->led_on ^= 1;
              HAL_GPIO_WritePin(
                  LD2_GPIO_Port,
                  LD2_Pin,
                  ctrl->led_on ? GPIO_PIN_SET : GPIO_PIN_RESET
              );
          }
      }
  }
  ```
### Step 7 — Main loop
* Replace `while(1)` with:
  ```
  while (1)
  {
      led_update(&led, system_tick_ms);
  }
  ```
## Full call chain
When TIM2 reaches its auto-reload value, this happens in hardware and software:
1. **TIM2 update event (hardware)**
  * The TIM2 counter overflows (ARR reached)
  * The STM32 hardware:
    * Sets the UIF (Update Interrupt Flag)
    * Requests an interrupt from the NVIC
2. **NVIC vectors to the ISR**
  * The Cortex-M core jumps to `TIM2_IRQHandler()`
    * *This function is auto-generated by CubeMX and lives in `Core/Src/stm32f4xx_it.c`*
    * It looks like this:
      ```
      void TIM2_IRQHandler(void)
      {
          HAL_TIM_IRQHandler(&htim2);
      }
      ```
3. **`HAL_TIM_IRQHandler()`**
   * This is HAL’s generic interrupt dispatcher
   * Checks why the interrupt fired
   * Clears the interrupt flag
   * Calls the weak callback
4. **`HAL_TIM_PeriodElapsedCallback()`**
  * In the HAL library, this function is defined as **weak**
  * So when HAL reaches this line: `HAL_TIM_PeriodElapsedCallback(htim);`
    * my function runs
  * 
