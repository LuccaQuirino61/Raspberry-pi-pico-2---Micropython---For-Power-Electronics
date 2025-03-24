# Raspberry Pi Pico 2 - MicroPython for Power Electronics

## Objectives

This project aims to implement and test the viability of PWM and ADC functions on the Raspberry Pi Pico 2 for Power Electronics applications using MicroPython. In this article, we will explore the best ways to implement, compare, and test PWM and ADC functions with interrupt request synchronization, measuring response time to determine whether MicroPython can handle high-frequency real-time control.

## Setting Up the Raspberry Pi Pico 2

This project will use the Thonny IDE to program the algorithms. To download it, visit: [Thonny IDE](https://thonny.org/)

### Steps to configure Thonny for MicroPython:

1. Connect the microcontroller to the computer while holding the **BOOTSEL** button.
2. Open Thonny and go to:
   - **Tools** → **Options** → **Interpreter** → **MicroPython (Raspberry Pi Pico)** → **Install or update MicroPython** → **OK**
3. Click the **Run** button to check if the MicroPython shell (REPL) is working correctly.

Once completed, you are ready to start programming your algorithm.

## Enabling PWM

To begin, let's understand how to activate PWM, its main functions, and parameters. First, we import the `machine` library, which contains the main functions of our microcontroller. For more details, visit: [MicroPython machine library](https://docs.micropython.org/en/latest/library/machine.html)

```python
from machine import PWM
```

This line imports the class that allows us to set the PWM for the RP2350.

### Understanding PWM Parameters We're going to use

- **`pin`**: GPIO pin used for PWM output.
- **`freq`**: Frequency of the PWM signal in Hertz (Hz). Higher values result in a faster switching rate.
- **`duty_u16`**: 16-bit duty cycle (0 to 65535), where 0 is always off and 65535 is always on.

### Creating a PWM Object

To activate PWM, use the following function:

```python
pwm = PWM(pin, freq=1000, duty_u16=32768)  # Create a PWM object on a pin
```

This function initializes a PWM signal by defining the pin, frequency, and 16-bit duty cycle.

Each parameter can also be set individually:

```python
pwm = PWM(21)  # Create a PWM object on pin 21
pwm.freq(1000)  # Set PWM frequency to 1000 Hz
pwm.duty_u16(32768)  # Set PWM duty cycle to 50%
```

To disable PWM at any time, use:

```python
pwm.deinit()  # Disable the PWM output
```

### Example: Implementing PWM to Control the Built-in LED Brightness

```python
from machine import Pin, PWM
import time

# Configure pin 25 (built-in LED) as a PWM output
led = PWM(Pin(25))   
led.freq(1000)  # Set PWM frequency to 1000 Hz

while True:
    # Increase brightness
    for duty in range(0, 65536, 500):  # Gradually vary duty cycle from 0% to 100%
        led.duty_u16(duty)  # Set duty cycle (16-bit: 0 to 65535)
        time.sleep(0.01)
    
    # Decrease brightness
    for duty in range(65535, -1, -500): 
        led.duty_u16(duty) 
        time.sleep(0.01)
    
    time.sleep(0.01)
```

### Explanation:
- **We added the `Pin` class from the `machine` library** to work with GPIOs.
- **We imported the `time` library** to manage delays.
- **The loop increases and decreases the LED brightness smoothly using PWM.**


