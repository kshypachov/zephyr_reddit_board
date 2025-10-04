# Zephyr ELI5: From a Newbie to Newbies — Часть 1: Создание кастомной платы

Привет, ребята! 👋  
Это первая статья из цикла «Zephyr ELI5: From a Newbie to Newbies», в которой я, такой же изучающий Zephyr человек, делюсь опытом. Мы будем пошагово разбираться с тем, как описывать кастомную плату: `.dts`, `Kconfig`, `board.yml` и всё, что нужно, чтобы Zephyr начал с ней работать.
Мы будем использовать плату от WeAct c чипом STM32F401CEU6. Я использую MacOS и Zephyr v4.2.0.

> ⚠️ Я не претендую на 100% правильность и строгость по стандартам. Если ты — профи в Zephyr, буду рад любой критике в комментариях.

---

## 💡 План серии
- **Статья №2** — Подключение W5500
- **Статья №1** — Создание описания платы: `dts`, `Kconfig`, `board.yml`.
- **(Возможно позже)** Статья №0 — Установка Zephyr SDK и настройка CLion IDE.

> Если эта статья будет полезна то будт следующие статьи
---

## 🧰 Что пригодится

- **STM32CubeMX или STM32CubeIDE** — очень удобны для настройки тактирования.
- **Zephyr SDK установлен** (в моем случае в `~/zephyrproject`)

---

## 📁 Структура проекта

В моём случае Zephyr установлен здесь:

```bash
/Users/kiro/
├── zephyrproject/zephyr/
└── zephyr-sdk-0.17.4/
```

---

## 📍 Где создавать плату

Внутри `zephyr/boards/arm` создаём новую директорию для платы:

```bash
cd ~/zephyrproject/zephyr/boards/arm
mkdir reddit_board
cd reddit_board
```

> Мы будем использовать имя нашей платы такое как и имя папки "reddit_board"


## 🧩 Шаг 1: Файл `Kconfig.reddit_board`

```bash
touch ~/zephyrproject/zephyr/boards/arm/reddit_board/Kconfig.reddit_board
```

Содержимое:

```kconfig
config BOARD_REDDIT_BOARD
	select SOC_STM32F401XE
```

> Название SOC берём из `soc/st/stm32/stm32f4x/Kconfig.soc` обратите вниминие что мы не указываем точное имя чипа, мы указываем серию. 
> То есть в нашем примере SOC_STM32F401XE это общее название для чипов STM32F401CE, STM32F401VE, STM32F401RE

> Имя файла формируется по такой формуле `Kconfig.BOARD_NAME` в нашем случае `Kconfig.reddit_board`
---

## 🧾 Шаг 2: Файл `board.yml`

```bash
touch ~/zephyrproject/zephyr/boards/arm/reddit_board/board.yml
```

Содержимое:

```yaml
board:
  name: reddit_board
  full_name: reddit_board_v1
  vendor: st
  socs:
    - name: stm32f401xe
```

- `name:` — должно совпадать с именем dts-файла и Kconfig
- `full_name:` — произвольное человекочитаемое имя
  - `socs:`  — тут мы перечисляем SoC которые присутствуют на плате, в нашем случае у нас только один SoC - такой же как и в фале Kconfig.reddit_board

---

## 🌲 Шаг 3: Файл `reddit_board.dts`

```bash
touch ~/zephyrproject/zephyr/boards/arm/reddit_board/reddit_board.dts
```

Содержимое:

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

> Тут нам как раз понадобится STM32CubeIDE или STM32CubeMX. Обе эти IDE имеют очень удобный интерфейс для настройки тактирования и подбора делителей 
> и множителей PLL и RCC

> В данном файле мы включили минимальный набор переферии, в следующих статьях мы будем добавлять блоки. 

> Формат имени файла BOARD_NAME.dts, в нашем случае `reddit_board.dts` 


---

## 🧩 Шаг 4: Файл `reddit_board_defconfig`

```bash
touch ~/zephyrproject/zephyr/boards/arm/reddit_board/reddit_board_defconfig
```

Содержимое:

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
> Это необязательный файл, без него работать будет. МЫ используем его ввиду его крайней удобности и полезности.

> Этот файл применяет конфигурацию по умолчанию для вашей платы при компиляции. Файл не является обязательным но является хорошим вариантом для укзания 
> необходимых для платы опций. Например тут мы указываем опции для запуска UART и Shell (интерактивная оболочка для взаимодействия с устройством).

---

## ✅ Проверка конфигурации

Активируйте виртуальное окружение:

```bash
source ~/zephyrproject/.venv/bin/activate
```

Проверьте, что ваша плата видна:

```bash
cd ~/zephyrproject/zephyr/samples/basic/blinky
west boards | grep reddit
# → reddit_board
```

---

## 🚀 Компиляция проекта

Перейдите в пример:

```bash
cd ~/zephyrproject/zephyr/samples/basic/blinky
```

Соберите:

```bash
west build -p always -b reddit_board
```

Результат:

```bash
 ~/zephyrproject/zephyr/samples/basic/blinky/build/zephyr/zephyr.hex
```

Зашейте прошивку на STM32F401CEU6 через DFU или ST-Link. Если всё верно — светодиод моргает, консоль доступна на UART1 (PA9/PA10) @ 115200.

---

## 🐞 Типичные ошибки

- `defined without a type`: Проверьте `select SOC_...` и правильность имени SOC
- `BOARD_REDDIT_BOARD` не начинается с `BOARD_`
- Плата не отображается в `west boards` — проверьте `board.yml` и пути
- Важно! board.yml не board.yaml

---

## 🧷 Заключение

Вы только что описали кастомную плату в Zephyr! Дальше — добавление Flash (например, W25Q128), SPI, I2C, и прочие периферийные устройства.

> Весь код и файлы можно найти в этом коммите: [GitHub ссылка]

Пишите в комменты, если хотите увидеть следующую статью раньше 😄
