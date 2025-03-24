# Raspberry Pi Pico 2 - MicroPython for Power Electronics

## Table of Contents  
1. [Objectives](#objectives)  
2. [Setting Up the Raspberry Pi Pico 2](#setting-up-the-raspberry-pi-pico-2)  
3. [Enabling PWM](#enabling-pwm)  
   - [PWM Parameters](#pwm-parameters)  
   - [Example: LED Brightness Control](#example-implementing-pwm-to-control-the-built-in-led-brightness)  
4. [Using ADC](#using-adc-analog-to-digital-converter)  
   - [ADC Limitations](#adc-limitations-and-latency)  
5. [PWM + ADC Synchronization](#setting-up-adc-readings-with-pwm)  
6. [Overclocking](#overclocking-the-rp2350)  
7. [Interrupt Synchronization](#synchronization-interruptions-between-adc-and-pwm)  
8. [Results](#results)  

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

### PWM Parameters

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

## Using ADC (Analog-to-Digital Converter)

Now, let's configure and read values from the ADC on the Raspberry Pi Pico 2. The ADC class is included in the `machine` library

### Understanding ADC Parameters

- **`ADC(n)`**: Defines which ADC channel to read (`n` = 0, 1, 2 for GPIO26, GPIO27, GPIO28).
- **`read_u16()`**: Reads the ADC value as a 16-bit integer (0 to 65535).

## Understanding the ADC on RP2350

The RP2350 microcontroller has a 12-bit ADC, but MicroPython scales the output to a 16-bit range (0 to 65535). Each ADC reading represents a voltage between 0V and 3.3V, meaning:

- **0** → 0V
- **65535** → 3.3V
- **32768** → ~1.65V (mid-range)

### ADC Limitations and Latency
- The ADC has a **conversion latency**, meaning it takes a few microseconds to complete each reading.
- The RP2350 ADC has an **input impedance** that can affect precision; use a low-impedance source or an external capacitor to improve accuracy.
- The ADC is **single-ended**, meaning it cannot measure negative voltages directly.

## Setting Up ADC Readings with PWM 

This example configures two ADC channels and synchronizes them with a 50 kHz PWM signal. The ADC values are stored in buffers, and the time taken for each conversion is measured.

### Example Code

```python
import machine
import utime

# Configure ADC inputs
adc1 = machine.ADC(1)  # ADC Channel 1 (GPIO27)
adc2 = machine.ADC(2)  # ADC Channel 2 (GPIO28)

# Configure PWM on pin 21
pwm = machine.PWM(machine.Pin(21))
pwm.freq(50_000)  # 50 kHz frequency
pwm.duty_u16(32768)  # 50% duty cycle

# Buffers for ADC readings
buffer_adc1 = []
buffer_adc2 = []
buffer_tempos = []

contador = 0
MAX_MEDIDAS = 100

# Interrupt triggered by PWM at the start of each cycle
def pwm_isr(timer):
    global contador

    if contador >= MAX_MEDIDAS:
        pwm.deinit()  # Disable PWM after 100 cycles
        return

    t_inicio = utime.ticks_us()  # Start time before ADC reading

    # Read both ADC channels
    valor_adc1 = adc1.read_u16()
    valor_adc2 = adc2.read_u16()

    t_fim = utime.ticks_us()  # End time after reading
    tempo_conversao = t_fim - t_inicio

    # Store readings and conversion time
    buffer_adc1.append(valor_adc1)
    buffer_adc2.append(valor_adc2)
    buffer_tempos.append(tempo_conversao)

    contador += 1

# Configure a Timer to trigger the ISR at the beginning of each PWM cycle
timer = machine.Timer()
timer.init(freq=50_000, mode=machine.Timer.PERIODIC, callback=pwm_isr)

# Wait for measurements to complete before displaying results
while contador < MAX_MEDIDAS:
    utime.sleep_us(1)  # Small delay to avoid excessive CPU usage

# Calculate average conversion time
tempo_medio = sum(buffer_tempos) / len(buffer_tempos)
print(f"Average ADC Conversion Time: {tempo_medio:.2f} µs")
```

## Explanation of the Code

1. **ADC Configuration:**
   - Uses ADC channels 1 and 2 (GPIO27 and GPIO28) to read analog signals.
   - Each reading is taken using `read_u16()`, which returns a 16-bit scaled value.

2. **PWM Configuration:**
   - The PWM signal on pin 21 operates at 50 kHz.
   - A **50% duty cycle** is set (half of 65535).

3. **Interrupt Synchronization:**
   - A **Timer** is used to call the `pwm_isr()` function at the start of each PWM cycle.
   - This ensures ADC readings are taken at the exact moment needed.

4. **Measurement Timing:**
   - The `utime.ticks_us()` function records the time before and after ADC readings to measure conversion latency.

5. **Data Collection:**
   - The ADC values and conversion times are stored in lists (`buffer_adc1`, `buffer_adc2`, `buffer_tempos`).
   - The script collects **100 samples** before stopping.

## Results

After running the code, I got an experimental result of 16.5us average for reading two ADC channels.

## Overclocking the RP2350

For power electronics applications, for example: real-time control of converters, the latency caused by the interpretation of the high-level language used can be an obstacle. Therefore, an acceptable solution is to overclock the processor. In this case, we can obtain a more interesting response time.
And to do this, simply add the following line of code at the beginning of the algorithm:
```python
machine.freq(300000000)  # 300 MHz
```
The base frequency of the RP2350 is **150MHz**. And increasing this clock frequency to 300MHz is possible.

### Important precautions!

Overclocking the processor can cause unwanted heat and noise. Therefore, ensure the temperature and output signals for your respective application. If necessary, add heat sinks and noise filters.

## Synchronization interruptions between ADC and PWM

### Warning!

For the next step of this article, we use the aid of an oscilloscope to visualize the real signal generated and measure the response times using the instrument itself.

### How RP2350 interrupts work:

In micropython, there is a function class called `.irq()` which is used to configure interrupts on GPIO pins. That has two arguments: `trigger` and `handler`;

- **`trigger`**: Defines what will trigger the interrupt;
- **`handler`**: Defines which function will be called if the interruption is requested;

The available parameters for the `trigger` argument are:
1. **`Pin.IRQ_RISING`**  
   - Triggers the interrupt when the signal changes from **LOW to HIGH** (rising edge).

2. **`Pin.IRQ_FALLING`**  
   - Triggers the interrupt when the signal changes from **HIGH to LOW** (falling edge).

3. **`Pin.IRQ_LOW_LEVEL`**  
   - Triggers the interrupt while the signal is **LOW** (low level).

4. **`Pin.IRQ_HIGH_LEVEL`**  
   - Triggers the interrupt while the signal is **HIGH** (high level).

*You can use the following values ​​or combinations of them using the bitwise operator* `|`

But note that the parameter is always of the 'Pin.' type. In this case, if we want to trigger an interrupt using the PWM as a parameter, we need to transfer its signal to a "sync pin". This sync pin will be defined as an input and will be physically connected to the GPIO where the PWM is being triggered.

```python
from machine import Pin, PWM

# Configure PWM on GPIO 21
pwm_pin = PWM(Pin(21))  # Set PWM output on pin 21
pwm_pin.freq(40000)  # Set carrier frequency to 40 kHz
pwm_pin.duty_u16(32768)  # 50% duty cycle

# Configure Sync Pin
sync_pin = Pin(12, Pin.IN)

```

Once you physically connect pins **21** and **12** (or any other GPIO of your choice), the PWM signal will be transferred to the `sync pin`.

## PWM Synchronized Interrupt - sync_pin

Now we can understand how to trigger an interrupt via PWM. The next step is to create a function to handle this interruption. And, within this function, we will have the ADC reading, which, in this case, will be synchronized with some PWM event.



