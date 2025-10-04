# Zephyr ELI5: От новичка — новичкам. Часть 1: Создание кастомной платы

Привет, ребята! 👋  
Это первый пост в серии **«Zephyr ELI5: От новичка — новичкам»**, где я — человек, который изучает Zephyr так же, как и вы — делюсь своим опытом. Мы будем шаг за шагом разбирать процесс описания кастомной платы в Zephyr: создание `.dts`, `Kconfig`, `board.yml` и всего, что нужно, чтобы ваша плата заработала с Zephyr.

Мы будем использовать плату WeAct с чипом **STM32F401CEU6**. Я работаю на **macOS** с **Zephyr v4.2.0**.

> ⚠️ Я не претендую на 100% правильность и полное соответствие официальным практикам. Если вы — эксперт в Zephyr, буду рад вашим комментариям!

---

## 💡 План серии

- **Часть 2** — Подключение Ethernet-чипа W5500  
- **Часть 1** — Создание описания платы: `dts`, `Kconfig`, `board.yml`
- **(Возможно позже)** Часть 0 — Установка Zephyr SDK и настройка CLion IDE

> Если этот пост окажется полезным, я напишу следующие статьи!

---

## 📚 Полезные официальные ресурсы

- 🧭 **Гайд по портированию плат**  
  https://docs.zephyrproject.org/latest/hardware/porting/board_porting.html

- 🌳 **Введение в DeviceTree**  
  https://docs.zephyrproject.org/latest/build/dts/intro.html#devicetree-intro

---

## 🧰 Что понадобится

- **STM32CubeMX** или **STM32CubeIDE** — очень удобно для настройки тактирования.
- **Установленный Zephyr SDK** (в моем случае — в `~/zephyrproject`)

---

## 📁 Структура проекта

У меня Zephyr установлен здесь:

```bash
/Users/kiro/
├── zephyrproject/zephyr/
└── zephyr-sdk-0.17.4/
```

---

## 📍 Где создать описание платы

Внутри `zephyr/boards/arm` создаём новую папку для платы:

```bash
cd ~/zephyrproject/zephyr/boards/arm
mkdir reddit_board
cd reddit_board
```

> Мы будем использовать одинаковое имя для папки и платы: `reddit_board`

---

## 🧩 Шаг 1: Файл `Kconfig.reddit_board`

```bash
touch ~/zephyrproject/zephyr/boards/arm/reddit_board/Kconfig.reddit_board
```

Содержимое:

```kconfig
config BOARD_REDDIT_BOARD
	select SOC_STM32F401XE
```

> Символ `SOC_STM32F401XE` берём из `soc/st/stm32/stm32f4x/Kconfig.soc`.  
> Это не полное название чипа, а «идентификатор» SoC семейства STM32F401 (CE/VE/RE и т. д.).  
> Этот символ включает поддержку нужного SoC в Zephyr.

> Формат имени файла: `Kconfig.BOARD_NAME`, в нашем случае `Kconfig.reddit_board`.

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

- `name:` — должно совпадать с именем `.dts` и `Kconfig`
- `full_name:` — произвольное человекопонятное имя
- `socs:` — список SoC, используемых на плате (в нашем случае один)

---

## 🌲 Шаг 3: Файл `reddit_board.dts`

```bash
touch ~/zephyrproject/zephyr/boards/arm/reddit_board/reddit_board.dts
```

Содержимое:

```dts
/dts-v1/; 													// Device Tree Source version 1 — обязательный заголовок
#include <st/f4/stm32f401xe.dtsi>							// Базовые определения SoC STM32F401xE
#include <st/f4/stm32f401c(d-e)ux-pinctrl.dtsi>				// Определения пин-контроля STM32F401C(D/E)Ux
#include <zephyr/dt-bindings/input/input-event-codes.h>		// Коды событий ввода (например, KEY_0)

/ {
	model = "Reddir v1 Board";								// Человекопонятное название платы
	compatible = "st,reddit_board";							// Строка для сопоставления в драйверах

	chosen {
		zephyr,console = &usart1;							// USART1 как системная консоль
		zephyr,shell-uart = &usart1;						// USART1 для оболочки
		zephyr,sram = &sram0;								// Основная SRAM
		zephyr,flash = &flash0;								// Основная Flash
	};

	leds {
		compatible = "gpio-leds";							// Узел для GPIO-светодиодов
		user_led: led {										// Метка для пользовательского светодиода
		gpios = <&gpioc 13 GPIO_ACTIVE_LOW>;			// Светодиод на PC13, активен по низкому уровню
			label = "User LED";								// Метка светодиода
		};
	};

	gpio_keys {
		compatible = "gpio-keys";							// Узел для кнопок
		user_button: button {								// Метка кнопки
		label = "KEY";									// Метка кнопки
			gpios = <&gpioa 0 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;	// Кнопка на PA0, активна по низкому уровню с подтяжкой
			zephyr,code = <INPUT_KEY_0>;					// Логический код, например KEY_0
		};
	};

	aliases {
		led0 = &user_led;									// Алиас led0
		sw0 = &user_button;									// Алиас sw0
	};
};

&usart1 {
	pinctrl-0 = <&usart1_tx_pa9 &usart1_rx_pa10>;			// Пины TX/RX USART1: TX=PA9, RX=PA10
	pinctrl-names = "default";								// Имя конфигурации pinctrl
	status = "okay";										// Включить периферию
	current-speed = <115200>;								// Скорость UART
};

&clk_hse {
	clock-frequency = <DT_FREQ_M(25)>;  					// Внешний кварц 25 МГц
	status = "okay";										// Включить HSE
};

&pll {
	div-m = <25>;											// Делитель входа PLL
	mul-n = <200>;											// Множитель PLL
	div-p = <2>;											// Делитель выхода PLL
	div-q = <7>;											// Делитель другого выхода PLL
	clocks = <&clk_hse>;									// Источник PLL — HSE
	status = "okay";										// Включить PLL
};

&rcc {
	clocks = <&pll>;										// Использовать PLL как системный источник
	clock-frequency = <DT_FREQ_M(100)>;						// Итоговая частота 100 МГц
	ahb-prescaler = <1>;									// Делитель AHB
	apb1-prescaler = <2>;									// Делитель APB1 (макс. 50 МГц)
	apb2-prescaler = <1>;									// Делитель APB2
};
```

> Здесь как раз пригодится STM32CubeIDE или STM32CubeMX — они упрощают подбор делителей и множителей PLL/RCC.  
> Этот `.dts` содержит минимальный набор периферии. В следующих статьях мы будем его расширять.  
> Формат имени файла: `BOARD_NAME.dts` — у нас `reddit_board.dts`.

📘 Официальный гайд по Devicetree:  
https://docs.zephyrproject.org/latest/build/dts/intro.html#devicetree-intro

---

## 🧬 Как работает цепочка наследования DTS (DeviceTree Chaining)

Когда вы включаете такой файл в `reddit_board.dts`:

```dts
#include <st/f4/stm32f401xe.dtsi>
```

Вы подключаете не только этот файл — вы запускаете **цепочку include**, которая подтягивает всё более специфичные определения оборудования для вашего SoC.

Цепочка выглядит так:

```
skeleton.dtsi
  └── armv7-m.dtsi
        └── stm32f4.dtsi
              └── stm32f401.dtsi
                    └── stm32f401xe.dtsi   ← мы включаем этот
```

Файлы находятся здесь:

```
zephyr/dts/arm/st/f4/
```

Каждый файл добавляет свой слой:

| Файл              | Назначение                                            |
|-------------------|-------------------------------------------------------|
| `skeleton.dtsi`   | Минимальные базовые определения для любого devicetree |
| `armv7-m.dtsi`    | Общие узлы для ARM Cortex-M (CPU, NVIC, SysTick и т. д.)|
| `stm32f4.dtsi`    | Общие узлы для всех STM32F4                           |
| `stm32f401.dtsi`  | Узлы для семейства STM32F401                          |
| `stm32f401xe.dtsi`| Адреса периферии, IRQ, часы для STM32F401xE           |

---

### 💡 Зачем это важно?

Потому что мы **не переопределяем всё с нуля**.  
Подключив `stm32f401xe.dtsi`, мы наследуем всё из родительских файлов: информацию о CPU, контроллер прерываний, базовую карту памяти, деревья тактирования и т. д.

Это позволяет сосредоточиться только на том, что **уникально для вашей платы**:

- GPIO для светодиодов и кнопок
- Назначения выводов
- Внешняя периферия (SPI-флеш, сенсоры)
- Источник тактирования и PLL

Можно думать об этом как о наследовании классов или слоистой конфигурации.

📘 Официальный гайд Devicetree:  
https://docs.zephyrproject.org/latest/build/dts/intro.html#devicetree-intro

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

> Этот файл необязателен, но крайне удобен!  
> Он задаёт дефолтные опции конфигурации для вашей платы при сборке. Здесь мы включили UART и оболочку — это очень полезно для отладки.

---

## ✅ Проверка конфигурации

Активируйте виртуальное окружение Python:

```bash
source ~/zephyrproject/.venv/bin/activate
```

Проверьте, что плата обнаруживается:

```bash
cd ~/zephyrproject/zephyr/samples/basic/blinky
west boards | grep reddit
# → reddit_board
```

---

## 🚀 Сборка примера

Перейдите в папку с примером:

```bash
cd ~/zephyrproject/zephyr/samples/basic/blinky
```

Соберите проект:

```bash
west build -p always -b reddit_board
```

Результат:

```bash
~/zephyrproject/zephyr/samples/basic/blinky/build/zephyr/zephyr.hex
```

Прошейте этот `.hex` в **STM32F401CEU6** через DFU или ST-Link.  
Если всё сделано правильно, светодиод будет мигать, а оболочка будет доступна на UART1 (PA9/PA10) @ 115200 бод.

---

## 🐞 Частые ошибки

- `defined without a type` — проверьте `select SOC_...` и правильность имени SoC
- `BOARD_REDDIT_BOARD` — должно начинаться с `BOARD_`
- Плата не видна в `west boards` — проверьте `board.yml` и пути
- Важно: это `board.yml`, **не** `board.yaml`!

---

## 🧷 Заключение

Вы только что создали собственное описание платы в Zephyr! 🎉  
Дальше — добавление внешней памяти W25Q128, SPI, I2C и других периферийных устройств.

> Все файлы для этой платы можно найти в этом коммите: [GitHub link]

📘 Официальный гайд по портированию плат:  
https://docs.zephyrproject.org/latest/hardware/porting/board_porting.html

Напишите в комментариях, если хотите увидеть следующую статью раньше 😄
