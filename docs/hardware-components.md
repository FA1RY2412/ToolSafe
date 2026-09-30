# Компоненти ToolSafe

Дослідження станом на 03.10.2026. Джерела нижче належать ST, TDK InvenSense та Wokwi. Для MPU6050 отримано індексовані фрагменти PDF виробника з номером документа, редакцією й сторінкою. Пряме завантаження старих посилань TDK повернуло redirect/404. Це обмежує повноту читання, тому до звіту треба додати доступну офіційну копію та перевірити решту сторінок.

## Наявна платформа ЛР1

| Вузол | Перевірено в локальних файлах | Первинне джерело |
|---|---|---|
| STM32C031C6 | `firmware/Makefile`: Cortex-M0+, `STM32C031xx`, linker STM32C031C6; `.ioc`: 48 МГц | [ST DS13867 Rev 4](https://www.st.com/resource/en/datasheet/stm32c031c6.pdf), с. 1, 9, 32–38 |
| USART2 | `main.c`: 115200 8N1; `stm32c0xx_hal_msp.c`: PA2 TX, PA3 RX, AF1; `diagram.json`: перехресне з'єднання з serial monitor | ST DS13867, таблиці pin/AF с. 32–38; [Wokwi C031C6](https://docs.wokwi.com/parts/board-st-nucleo-c031c6) |
| LED | `main.h`: PA5, `main.c`: `HAL_GPIO_TogglePin` і `HAL_Delay(1000)` | [Wokwi C031C6, Onboard LED](https://docs.wokwi.com/parts/board-st-nucleo-c031c6#onboard-led), LED LD4 активний HIGH |
| I²C1 | PB8 SCL, PB9 SDA, AF6; timing `0x20303E5D`; датчика на схемі немає | ST DS13867, с. 34, 37–38; `.ioc` та `stm32c0xx_hal_msp.c` |

Ініціалізація I²C в коді не підтверджує обмін або частоту SCL. Частоту 100 кГц заплановано для ЛР2, її треба перевірити за clock tree, timing і осцилограмою. DMA не обираємо основою плану: [документація Wokwi](https://docs.wokwi.com/parts/board-st-nucleo-c031c6#simulation-features) позначає DMA цієї моделі як не реалізований.

## Заплановані сенсори ЛР2

| Сенсор | Сигнали та інтерфейс | Дані й роль | Живлення, регістри, невідоме |
|---|---|---|---|
| `wokwi-pir-motion-sensor` | VCC, GND, OUT; GPIO, кандидат PA1 | OUT 0→1 означає підхід, FR-04 | Реальний модуль і його виробник ще не обрані; напруга VCC, рівень OUT і струм фізичного аналога не підтверджені. Register map не застосовується до GPIO |
| `wokwi-photoresistor-sensor` | VCC, GND, AO; ADC, кандидат PA0/ADC_IN0; DO не використовуємо | Відносна зміна ADC, FR-05. Модель містить LDR і 10 кОм дільник | Проєктне живлення дільника 3.3 В, AO повинен лишатися в діапазоні ADC. Для фізичного LDR модель, струм модуля та datasheet ще невідомі. Register map відсутній |
| MPU6050, `wokwi-mpu6050` | VCC, GND, SCL→PB8, SDA→PB9, AD0→GND; I²C 7-bit 0x68, план 100 кГц | Прискорення в g та кутова швидкість у °/с, FR-06 | Для чіпа VDD 2.375–3.46 В; типовий струм accel+gyro без DMP 3.8 мА. Breakout має власну схему, його VCC не слід прирівнювати до VDD чіпа |

Піни PA0/PA1 є кандидатами, зміни `.ioc` та `diagram.json` ще не виконані. Перед реалізацією звірити розводку обраної плати та конфлікти. Немає підтвердження, що всі фізичні модулі можна живити з 3V3 Nucleo: спочатку потрібні їхні datasheet, схема breakout та сумарний струм.

PIR у моделі тримає OUT HIGH 5 с, потім ігнорує рух 1.2 с. Це параметри моделі, а не строк реакції прошивки. Вона має фіксувати фронт, не кожне опитування HIGH. [Wokwi PIR, Pin names/Attributes/Using the sensor](https://docs.wokwi.com/parts/wokwi-pir-motion-sensor).

Світло змінює AO дільника. Таблиця Wokwi наведена для VCC=5 В та Arduino ADC; її числові значення не переносимо до STM32. Збережемо raw ADC і перевіримо напрямок зміни. Люкс не обіцяємо без калібрування. [Wokwi LDR, Pin names/Operation](https://docs.wokwi.com/parts/wokwi-photoresistor-sensor). Для STM32 діапазон визначається VDDA/VREF, а не позначкою цифрової 5V-tolerance; див. [ST DS13867](https://www.st.com/resource/en/datasheet/stm32c031c6.pdf), с. 20, розділ ADC characteristics.

## MPU6050, первинні PDF

[PS-MPU-6000A-00 Rev 3.4](https://invensense.tdk.com/wp-content/uploads/2015/02/MPU-6000-Datasheet.pdf), с. 14, Electrical and Other Common Specifications, підтверджує VDD та струм вище. [Wokwi MPU6050](https://docs.wokwi.com/parts/wokwi-mpu6050) підтверджує 0x68 при AD0 LOW і 0x69 при AD0 HIGH, контакти та одиниці моделі. Для HAL API з адресою у старших 7 бітах плануємо `0x68 << 1`; це параметр API, адреса самого чіпа залишається 0x68.

[RM-MPU-6000A-00 Rev 4.2](https://invensense.tdk.com/wp-content/uploads/2015/02/MPU-6000-Register-Map.pdf):

| Дія в майбутньому драйвері | Регістр | Сторінка PDF та стан |
|---|---|---|
| Перевірити ідентифікатор | WHO_AM_I 0x75, очікуване 0x68 незалежно від AD0 | с. 45, підтверджено фрагментом виробника |
| Вивести зі sleep | PWR_MGMT_1 0x6B, reset 0x40, SLEEP bit 6 | с. 8, підтверджено таблицею виробника; повну процедуру запуску ще звірити |
| Читати прискорення узгодженим burst | ACCEL_XOUT_H…ZOUT_L 0x3B…0x40, signed 16-bit | с. 29, для ±2 g чутливість 16384 LSB/g підтверджена; початковий діапазон ±2 g треба оцінити щодо насичення під час удару |
| Читати кутову швидкість | GYRO_XOUT_H…ZOUT_L 0x43…0x48 | с. 31, потребує звірення повної копії PDF перед реалізацією |
| Обрати діапазони, фільтр і sample rate | ACCEL_CONFIG, GYRO_CONFIG, CONFIG, SMPLRT_DIV | Значення конфігурації не реалізовані; уточнити за повним Register Map |

Підтяжки SCL/SDA, VLOGIC і живлення breakout треба перевірити за схемою конкретного модуля. У симуляції перевірятимемо транзакції, дані, NACK і затримку. Висновок про механічну подію є правилом ToolSafe, а не функцією самого MPU6050.

## Що ще потрібно для дослідження за PDF ЛР1

Методичка вимагає офіційний datasheet із конкретними сторінками для кожного сенсора, pinout, живлення/струм та register map, де він існує. Wokwi є першоджерелом для моделі, але не datasheet фізичного PIR або LDR. Обрати фізичні аналоги й додати перевірені PDF виробників та бюджет живлення. Не приписувати безіменному PIR характеристики HC-SR501 або називати LDR GM5528 без підтвердження.
