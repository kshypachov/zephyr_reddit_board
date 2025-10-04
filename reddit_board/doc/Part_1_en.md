# Zephyr ELI5: From a Newbie to Newbies — Part 1: Creating a Custom Board

Hey folks! 👋  
This is the first post in a series called **"Zephyr ELI5: From a Newbie to Newbies"**, where I — someone who's learning Zephyr just like you — share my experience. We'll go step-by-step through the process of describing a custom board in Zephyr: creating `.dts`, `Kconfig`, `board.yml`, and everything needed to make your board work with Zephyr.

We'll be using a WeAct board with the **STM32F401CEU6** chip. I'm working on **macOS** with **Zephyr v4.2.0**.

> ⚠️ I don’t claim to be 100% correct or fully compliant with official best practices. If you're a Zephyr expert — I'd love your feedback in the comments!

---

## 💡 Planned Series

- **Part 2** — Connecting the W5500 Ethernet chip  
- **Part 1** — Creating a board definition: `dts`, `Kconfig`, `board.yml`
- **(Maybe later)** Part 0 — Installing Zephyr SDK and configuring CLion IDE

> If this post is helpful, I’ll publish the next articles!

---


## 📚 Useful Official Resources

- 🧭 **Board Porting Guide**  
  https://docs.zephyrproject.org/latest/hardware/porting/board_porting.html

- 🌳 **DeviceTree Intro Guide**  
  https://docs.zephyrproject.org/latest/build/dts/intro.html#devicetree-intro

---

## 🧰 What You’ll Need

- **STM32CubeMX** or **STM32CubeIDE** — super helpful for clock configuration.
- **Zephyr SDK installed** (in my case, at `~/zephyrproject`)

---

## 📁 Project Structure

In my case, Zephyr is installed here:

```bash
/Users/kiro/
├── zephyrproject/zephyr/
└── zephyr-sdk-0.17.4/
```

---

## 📍 Where to Create Your Board

Inside `zephyr/boards/arm`, create a new folder for your board:

```bash
cd ~/zephyrproject/zephyr/boards/arm
mkdir reddit_board
cd reddit_board
```

> We'll use the same name for our board and folder: `reddit_board`

---

## 🧩 Step 1: `Kconfig.reddit_board` File

```bash
touch ~/zephyrproject/zephyr/boards/arm/reddit_board/Kconfig.reddit_board
```

Contents:

```kconfig
config BOARD_REDDIT_BOARD
	select SOC_STM32F401XE
```

> You get the `SOC_STM32F401XE` name from `soc/st/stm32/stm32f4x/Kconfig.soc`  
> This is not the full chip name — it's a generic name for STM32F401CE, STM32F401VE, STM32F401RE, etc.

> File naming format is: `Kconfig.BOARD_NAME`, so here it is `Kconfig.reddit_board`.

---

## 🧾 Step 2: `board.yml` File

```bash
touch ~/zephyrproject/zephyr/boards/arm/reddit_board/board.yml
```

Contents:

```yaml
board:
  name: reddit_board
  full_name: reddit_board_v1
  vendor: st
  socs:
    - name: stm32f401xe
```

- `name:` — should match the `.dts` filename and `Kconfig`
- `full_name:` — any human-readable name
- `socs:` — list of SoCs used on the board (we only have one here)

---

## 🌲 Step 3: `reddit_board.dts` File

```bash
touch ~/zephyrproject/zephyr/boards/arm/reddit_board/reddit_board.dts
```

Contents:

```dts
/dts-v1/; 													// Device Tree Source version 1 — required header for DTS files
#include <st/f4/stm32f401xe.dtsi>							// Include the SoC-specific base definitions for STM32F401xE
#include <st/f4/stm32f401c(d-e)ux-pinctrl.dtsi>				// Include pin control definitions for STM32F401C(D/E)Ux series
#include <zephyr/dt-bindings/input/input-event-codes.h>		// Include input event key codes (e.g., KEY_0)

/ {
	model = "Reddir v1 Board";								// Human-readable model name of the board
	compatible = "st,reddit_board";							// Compatible string used for matching in drivers or overlays

	chosen {
		zephyr,console = &usart1;							// Set USART1 as the system console (e.g., printk/log output)
		zephyr,shell-uart = &usart1;						// Use USART1 for shell interface (if enabled)
		zephyr,sram = &sram0;								// Define main SRAM region
		zephyr,flash = &flash0;								// Define main flash region
	};

	leds {
		compatible = "gpio-leds";							// Node for GPIO-controlled LEDs
		user_led: led {										// Define a label for the user LED
		gpios = <&gpioc 13 GPIO_ACTIVE_LOW>;			// LED is on GPIOC pin 13, active low
			label = "User LED";								// Human-readable label for the LED
		};
	};

	gpio_keys {
		compatible = "gpio-keys";							// Node for GPIO button inputs (key events)
		user_button: button {								// Define a label for the user button
		label = "KEY";									// Human-readable label
			gpios = <&gpioa 0 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;	// Button on GPIOA pin 0, active low with pull-up
			zephyr,code = <INPUT_KEY_0>;					// Logical input code, like KEY_0
		};
	};

	aliases {
		led0 = &user_led;									// Alias 'led0' used by Zephyr subsystems (e.g., samples)
		sw0 = &user_button;									// Alias 'sw0' used for button handling (e.g., in samples or input drivers)
	};
};

&usart1 {
	pinctrl-0 = <&usart1_tx_pa9 &usart1_rx_pa10>;			// Define the TX/RX pins for USART1: TX = PA9, RX = PA10
	pinctrl-names = "default";								// Define pinctrl configuration name (required)
	status = "okay";										// Enable this peripheral in the build
	current-speed = <115200>;								// Set UART baudrate to 115200
};

&clk_hse {
	clock-frequency = <DT_FREQ_M(25)>;  					// Use external crystal with 25 MHz frequency
	status = "okay";										// Enable HSE (High-Speed External) oscillator
};

&pll {
	div-m = <25>;											// PLL input divider
	mul-n = <200>;											// PLL multiplier
	div-p = <2>;											// PLL output divider for main system clock
	div-q = <7>;											// PLL output divider for main system clock
	clocks = <&clk_hse>;									// PLL source ( in this exaple - use external oscillator (HSE))
	status = "okay";										// Enable PLL
};

&rcc {
	clocks = <&pll>;										// Use PLL as system clock source
	clock-frequency = <DT_FREQ_M(100)>;						// Final core system clock frequency after applying PLL: 100 MHz
	ahb-prescaler = <1>;									// Division on AHB bus
	apb1-prescaler = <2>;									// Divide APB1 clock by 2 (max 50 MHz for STM32F4)
	apb2-prescaler = <1>;									// Division on APB2
};
```

> This is where STM32CubeIDE / STM32CubeMX is useful — they make it easy to configure clocks, PLL dividers and multipliers.

> This `.dts` defines the minimum required peripherals. We'll expand on it in future posts.

> Filename format: `BOARD_NAME.dts` — so here it’s `reddit_board.dts`

For more information about Devicetree syntax and structure, see the official guide:  
📘 https://docs.zephyrproject.org/latest/build/dts/intro.html#devicetree-intro

---

## 🧩 Step 4: `reddit_board_defconfig` File

```bash
touch ~/zephyrproject/zephyr/boards/arm/reddit_board/reddit_board_defconfig
```

Contents:

```kconfig
# SPDX-License-Identifier: Apache-2.0

CONFIG_ARM_MPU=y
CONFIG_HW_STACK_PROTECTION=y

CONFIG_SERIAL=y
CONFIG_UART_INTERRUPT_DRIVEN=y
CONFIG_CONSOLE=y
CONFIG_UART_CONSOLE=y

CONFIG_GPIO=y

CONFIG_SHELL=y
CONFIG_KERNEL_SHELL=y
CONFIG_SHELL_BACKEND_SERIAL=y
```

> This file is optional — but highly recommended!

> It sets the default config options for your board when you build for it. Here we enable UART and the shell interface — super useful for debugging.

---

## ✅ Verify Board Configuration

Activate the Python virtual environment:

```bash
source ~/zephyrproject/.venv/bin/activate
```

Check if the board is detected:

```bash
cd ~/zephyrproject/zephyr/samples/basic/blinky
west boards | grep reddit
# → reddit_board
```

---

## 🚀 Build a Sample

Go to the sample project:

```bash
cd ~/zephyrproject/zephyr/samples/basic/blinky
```

Build the project:

```bash
west build -p always -b reddit_board
```

Output file:

```bash
~/zephyrproject/zephyr/samples/basic/blinky/build/zephyr/zephyr.hex
```

Flash this `.hex` to your **STM32F401CEU6** using DFU or ST-Link.  
If everything worked, the LED will blink and a shell will be available on UART1 (PA9/PA10) @ 115200 baud.

---

## 🐞 Common Errors

- `defined without a type` — check your `select SOC_...` and make sure the name is valid
- `BOARD_REDDIT_BOARD` — must start with `BOARD_`
- Board not visible in `west boards` — check `board.yml` and file paths
- Be careful: it's `board.yml`, **not** `board.yaml`!

---

## 🧷 Final Notes

You’ve just created your own custom board definition in Zephyr! 🎉  
Next up: adding W25Q128 flash, SPI, I2C and other peripherals.

> You can find all the files for this board in this commit: [GitHub link , https://github.com/kshypachov/zephyr_reddit_board/edit/main/reddit_board/]

📘 Official porting guide:  
https://docs.zephyrproject.org/latest/hardware/porting/board_porting.html


Leave a comment if you want the next part of the series sooner 😄
