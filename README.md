# step_3_led_blink_with_modes
Building from step 2, adding user-button control over different modes

### Using these versions:
* `STM32CubeIDE` : `1.18.1`
* `STM32CubeMX` : `6.14.1`

### What we need to add
* A mode transition trigger
  * Without blocking
  * Without polling delays
  * The correct next trigger is the user button (B1) on the Nucleo
* Debounce the button

## Goals
* A tick-driven scheduler
* A software state machine
* Event-driven mode transitions
* Proper ISR-to-main communication

## Step 1 - Configure the button interrupt in CubeMX
In my `.ioc`:

### GPIO
* **`PC13-ANTI_TAMP`**
  * **Mode** : "External Interrupt Mode with Falling edge trigger detection"
  * **Pull-up/Pull-down** : "No pull-up and no pull-down"

### NVIC
* Enable "EXTI line [15:10] interrupts"

**Save to generate code**

## Step 2 - Add a mode-change request flag
* Add the following in the global "Private Variables section":
  ```
  volatile uint8_t  mode_change_requested = 0;
  ```

## Step 3 - Button ISR
* **This is not where mode is changed!**
* CubeMX already generated the EXTI handler
* Add this callback:
  ```
  void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
  {
      if (GPIO_Pin == B1_Pin)
      {
          mode_change_requested = 1;
      }
  }
  ```
* ISR sets a flag only

## Step 4 - Apply the mode change in main loop
* Modify `while(1)`:
  ```
  while (1)
  {
      if (mode_change_requested)
      {
          mode_change_requested = 0;
  
          led.mode++;
          if (led.mode > LED_MODE_FAST)
          {
              led.mode = LED_MODE_OFF;
          }
      }
  
      led_update(&led, system_tick_ms);
  }
  ```
## Step 5 - Debounce the button
We want...
* One **clean button event** per press
* Immunity to mechanical bounce (~5–20 ms)

### ISR
  * Latches “something happened”
  * Captures a timestamp

### Main loop
  * Waits for debounce time to expire
  * Confirms press
  * Emits a single event

### Button debounce state
* Remove all references to `mode_change_requested`
* Add the following in the global "Private Variables section":
  ```
  typedef struct
  {
      uint8_t  pending;
      uint32_t timestamp_ms;
  } button_db_t;
  
  volatile button_db_t button = {0};
  ```
### Update the EXTI callback
* Replace the existing EXTI callback with...
  ```
  void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
  {
      if (GPIO_Pin == B1_Pin)
      {
          if (!button.pending)
          {
              button.pending = 1;
              button.timestamp_ms = system_tick_ms;
          }
      }
  }
  ```
* This moves the ISR towards "timestamp-only" functionality; **it does no decision-making**

### Debounce handler (main loop)
* Add this function:
  ```
  #define DEBOUNCE_MS 50
  
  uint8_t button_debounce_update(void)
  {
      if (button.pending)
      {
          if ((system_tick_ms - button.timestamp_ms) >= DEBOUNCE_MS)
          {
              button.pending = 0;
  
              if (HAL_GPIO_ReadPin(B1_GPIO_Port, B1_Pin) == GPIO_PIN_RESET)
              {
                  return 1;  // Valid press
              }
          }
      }
      return 0;
  }
  ```

### Integrate into main loop
Modify `while(1)`:
```
while (1)
{
    if (button_debounce_update())
    {
        led.mode++;
        if (led.mode > LED_MODE_FAST)
        {
            led.mode = LED_MODE_OFF;
        }
    }

    led_update(&led, system_tick_ms);
}
```



## Expected behavior
* Each button press cycles:
  ```
  OFF → SLOW → FAST → OFF → ...
  ```
* No delays; no blocking; no ISR logic pollution
