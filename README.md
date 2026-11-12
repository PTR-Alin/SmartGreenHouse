# SmartGreenHouse
A smart greenhouse monitoring system developed by a team of four students, using sensors to track environmental conditions and determine optimal parameters for plant growth.

## STM32 Example (HAL-based)

Below is a compact example for an STM32 development board that reads common greenhouse inputs and controls outputs:

- **Temperature/Humidity** (example: DHT22 or similar digital sensor)
- **Soil moisture** (ADC input)
- **Light level** (ADC input)
- **Fan / pump / grow light** (GPIO outputs via relay or transistor stage)

```c
#include "main.h"
#include <stdio.h>
#include <string.h>

ADC_HandleTypeDef hadc1;
UART_HandleTypeDef huart2;

// GPIO pins (adapt to your board wiring)
#define FAN_GPIO_Port       GPIOA
#define FAN_Pin             GPIO_PIN_0
#define PUMP_GPIO_Port      GPIOA
#define PUMP_Pin            GPIO_PIN_1
#define GROW_LIGHT_GPIO_Port GPIOA
#define GROW_LIGHT_Pin      GPIO_PIN_2

// Thresholds (tune for your plants)
#define TEMP_MAX_C          30.0f
#define SOIL_DRY_ADC        2500U
#define LIGHT_LOW_ADC       1500U

// ---- Sensor stubs (replace with real drivers) ----
static float ReadTemperatureC(void) {
    // Replace with your sensor read function (I2C/1-Wire/GPIO protocol)
    return 28.5f;
}

static float ReadHumidityPct(void) {
    // Replace with your sensor read function
    return 62.0f;
}

static uint16_t ReadAdcChannel(uint32_t channel) {
    ADC_ChannelConfTypeDef sConfig = {0};
    sConfig.Channel = channel;
    sConfig.Rank = ADC_REGULAR_RANK_1;
    sConfig.SamplingTime = ADC_SAMPLETIME_15CYCLES;
    HAL_ADC_ConfigChannel(&hadc1, &sConfig);

    HAL_ADC_Start(&hadc1);
    HAL_ADC_PollForConversion(&hadc1, HAL_MAX_DELAY);
    uint16_t value = (uint16_t)HAL_ADC_GetValue(&hadc1);
    HAL_ADC_Stop(&hadc1);
    return value;
}

static void SetActuator(GPIO_TypeDef *port, uint16_t pin, GPIO_PinState state) {
    HAL_GPIO_WritePin(port, pin, state);
}

static void DebugPrint(const char *msg) {
    HAL_UART_Transmit(&huart2, (uint8_t *)msg, (uint16_t)strlen(msg), HAL_MAX_DELAY);
}

int main(void) {
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_ADC1_Init();
    MX_USART2_UART_Init();

    char line[128];

    while (1) {
        float tempC = ReadTemperatureC();
        float humidity = ReadHumidityPct();
        uint16_t soilAdc = ReadAdcChannel(ADC_CHANNEL_1);   // Soil moisture sensor
        uint16_t lightAdc = ReadAdcChannel(ADC_CHANNEL_2);  // LDR/light sensor

        // Control logic
        SetActuator(FAN_GPIO_Port, FAN_Pin, (tempC > TEMP_MAX_C) ? GPIO_PIN_SET : GPIO_PIN_RESET);
        SetActuator(PUMP_GPIO_Port, PUMP_Pin, (soilAdc > SOIL_DRY_ADC) ? GPIO_PIN_SET : GPIO_PIN_RESET);
        SetActuator(GROW_LIGHT_GPIO_Port, GROW_LIGHT_Pin, (lightAdc < LIGHT_LOW_ADC) ? GPIO_PIN_SET : GPIO_PIN_RESET);

        snprintf(line, sizeof(line),
                 "T=%.1fC H=%.1f%% Soil=%u Light=%u\r\n",
                 tempC, humidity, soilAdc, lightAdc);
        DebugPrint(line);

        HAL_Delay(2000); // sample every 2 seconds
    }
}
```

### Notes
- Generate board init code (`MX_GPIO_Init`, `MX_ADC1_Init`, `MX_USART2_UART_Init`) using **STM32CubeMX/STM32CubeIDE**.
- Use proper signal conditioning/protection when connecting sensors and relays.
- Calibrate ADC thresholds (`SOIL_DRY_ADC`, `LIGHT_LOW_ADC`) using your real sensors.
