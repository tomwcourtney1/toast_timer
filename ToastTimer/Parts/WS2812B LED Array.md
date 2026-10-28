The LED array is 16 LEDs in an 8x8 grid, each with a WS2812B control chip ([datasheet](docs/WS2812B.pdf)). 

#### WS2812B Power Characteristics

| Chracteristic                                          | Value           |
| ------------------------------------------------------ | --------------- |
| **Power Input ($V_{DD}$)**                             | $3.5V$ - $5.3V$ |
| **Min Input (Control) Voltage High ($V_{IH}$)**   <br> | $0.7V_{DD}$     |
| **Max Input (Control) Voltage Low ($V_{IL}$)**   <br>  | $0.3V_{DD}$     |
| **Current Draw**                                       | Small           |
#### LED Power Characteristics

Taken from the product page: [Amazon](https://www.amazon.com.au/dp/B0F1CTXJZ4?ref=ppx_yo2ov_dt_b_fed_asin_title)

| Characteristic        | Value                            |
| --------------------- | -------------------------------- |
| **Input Voltage**<br> | $5VDC$                           |
| **Power** <br>        | $19.2W$                          |
| **Current**           | $Power/Voltage = 19.2/5 = 3.84A$ |
#### Communication Protocol

Each LED chip can consume a 3Byte packet, where each byte corresponds to Green, Red, Blue. 

`[B0, B1, B2] => [G,R,B]`

So if you want the LED to be white you send `[0xFF, 0xFF, 0xFF]`. If you want it to be blue you send `[0x00, 0x00, 0xFF]`. Not sure why the manufacturer deviated from `RGB` for `GRB` but oh well.

Each bit is encoded with voltage high and low for both 0 and 1. The duration of the voltage high and low dictates if the signal is communicating a 1 or 0.

| Bit Value | High or Low | Duration            | Voltage  |
| --------- | ----------- | ------------------- | -------- |
| 0         | High        | $0.4us$ ($T_{0H}$)  | $V_{IH}$ |
| 0         | Low         | $0.85us$ ($T_{0L}$) | $V_{IL}$ |
| 1         | High        | $0.8us$ ($T_{1H}$)  | $V_{IH}$ |
| 1         | Low         | $0.45us$ ($T_{1L}$) | $V_{IL}$ |
**Example: Send a 0**

![[Pasted image 20261028154728.png]]

**Example: Send a 1**

![[Pasted image 20261028154735.png]]

#### Hardware Layer

So what hardware is the best way to send 1s and 0s as described above if you're using a run of the mill STM32? We can see that the width of the pulse dictates the value being sent so perhaps the Pulse Width Modulation (PWM) peripheral is appropriate. 

**PWM Frequency**

First thing to note is that for both a 0 and 1 the total width of the cycle is $1.25us$, therefore the frequency of the control signal is

$f = 1 / \lambda$ 
$\therefore$ $f = \frac{1}{1.25 \times 10^-6} = 800,000Hz$ 

So your PWM needs to be able to output a $800KHz$ signal and you probably want tooling to be able to verify that is the case.

**Configuring the PWM**

The WS2812B protocols cycle duration is $1.25us$ a minimum resolution of $0.05us$ (or $50ns$) is necessary. Thus we require a timer that can output with a resolution of $50ns$ which is a frequency of $20MHz$. 

For the **STM32F446** micro, `TIMER 1` can act as a `PWM`. It's attached to the APB2 bus which according to the [datasheet][docs\DS_stm32f446mc.pdf] has a max clock frequency of $180MHz$. Much faster than the `20MHz` required. 

![[Pasted image 20261028162836.png|250]]

So, we scale down the `TIMER 1` operating frequency with the `TIMER 1` pre-scaler register value `TIM1_PSC`.  

Timer Frequency = $\frac{f_{TIM1CLK}}{TIM1\_PSC + 1} = \frac{180MHz}{8 + 1} = 20MHz$  

Thus set `TIM1_PSC` to 8 to have each `TIMER 1` tick duration be $50ns$. So then for 1 cycle of the 800KHz PWM we need 1.25/0.05 ticks = 25 ticks.

At this point we introduce the Auto Reload Register (ARR) and the Capture/Compare Register (CCR). The ARR value tells TIM1 the total pulse width so it will be set to 24 (0 based counting) to get 1.25us pulse width.

The CCR tells the TIM1 PWM when to go from high to low. So the CRR/ARR determines the duty cycle of the PWM signal. For example to transmit a 0 the high voltage time is 0.4us which is 8x 0.05us (50ns ticks). So to send a 0 to the WS2812B we set CCR to 8. Similarly to send 1 to the WS2812B we send set the CCR to 16 as it's width is 0.8us. 

TIM1 has a register called CNT which is literally just the number of ticks since last reset and it's always comparing that to ARR to know when to begin again. Similarly when configured to a PWM it's comparing CNT to CCR to know when to go from high to low and control the duty cycle.




**Direct Memory Access**

How to actually modulate the pulse in an efficient manner that doesn't muck with your main software loop. Direct Memory Access is appropriate here. At a high level you load the bytes representing the color for the LED into the DMA and the PWM will pull it out of there without using any CPU cycles (or very little anyway).

1. Configure the PWM to pull from the DMA
2. Stash your desired color values in DMA
3. Turn on the PWM and it will output what's in the DMA
4. Turn off the PWM when it's finished outputting the DMA values








