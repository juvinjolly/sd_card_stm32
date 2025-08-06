#include "main.h"
#include "fatfs.h"

#include <stdio.h>
#include <string.h>
#include <stdarg.h>

SPI_HandleTypeDef hspi1;
UART_HandleTypeDef huart2;

void SystemClock_Config(void);
static void MX_GPIO_Init(void);
static void MX_SPI1_Init(void);
static void MX_USART2_UART_Init(void);
FRESULT READ_sdcard(void);
FRESULT WRITE_sdcard(const char *data);

void myprintf(const char *fmt, ...);

void myprintf(const char *fmt, ...) {
  static char buffer[256];
  va_list args;
  va_start(args, fmt);
  vsnprintf(buffer, sizeof(buffer), fmt, args);
  va_end(args);

  int len = strlen(buffer);
  HAL_UART_Transmit(&huart2, (uint8_t*)buffer, len, HAL_MAX_DELAY);
}

BYTE readBuf[300];
FATFS FatFs; 	// Fatfs handle
FIL fil; 		// File handle
FRESULT fres;   // Result after operations

char GPS_log[256];  // Buffer to hold GPS data

int main(void)
{
  HAL_Init();
  SystemClock_Config();

  MX_GPIO_Init();
  MX_SPI1_Init();
  MX_USART2_UART_Init();
  MX_FATFS_Init();

  HAL_Delay(5000); // Wait for SD card to settle

  // Mount filesystem
  fres = f_mount(&FatFs, "", 1);
  if (fres != FR_OK)
  {
    myprintf("f_mount error (%i)\r\n", fres);
    while(1);
  }

  // Check free space
  DWORD free_clusters, free_sectors, total_sectors;
  FATFS* getFreeFs;

  fres = f_getfree("", &free_clusters, &getFreeFs);
  if (fres != FR_OK)
  {
    myprintf("f_getfree error (%i)\r\n", fres);
    while(1);
  }

  total_sectors = (getFreeFs->n_fatent - 2) * getFreeFs->csize;
  free_sectors = free_clusters * getFreeFs->csize;

  myprintf("SD card stats:\r\n%10lu KiB total drive space.\r\n%10lu KiB available.\r\n",
           total_sectors / 2, free_sectors / 2);

  // Read existing file contents (optional)
  READ_sdcard();

  // Example GPS data (replace this with your GPS data input)
  strcpy(GPS_log, "$GPGGA,123519,4807.038,N,01131.000,E,1,08,0.9,545.4,M,46.9,M,,*47");

  // Write GPS data to SD card
  fres = WRITE_sdcard(GPS_log);
  if (fres == FR_OK) 
  {
    myprintf("GPS data logged successfully.\r\n");
  } else
  {
    myprintf("Failed to log GPS data (%i)\r\n", fres);
  }

  while (1)
  {
    // Your main loop code
  }
}

FRESULT READ_sdcard(void)
{
    fres = f_open(&fil, "myfile.txt", FA_READ | FA_OPEN_EXISTING);
    if (fres != FR_OK) {
        myprintf("f_open error (%i)\r\n", fres);
        return fres;
    }

    myprintf("Opened 'myfile.txt' for reading:\r\n");

    while (f_gets((TCHAR*)readBuf, sizeof(readBuf), &fil)) {
        myprintf("%s", readBuf);
    }

    fres = f_close(&fil);
    if (fres != FR_OK) {
        myprintf("f_close error (%i)\r\n", fres);
    }

    return fres;
}

FRESULT WRITE_sdcard(const char *data)
{
    fres = f_open(&fil, "myfile.txt", FA_WRITE | FA_OPEN_ALWAYS);
    if (fres != FR_OK)
    {
        myprintf("f_open error (%i)\r\n", fres);
        return fres;
    }

    fres = f_lseek(&fil, f_size(&fil)); // Move to end to append
    if (fres != FR_OK)
    {
        myprintf("f_lseek error (%i)\r\n", fres);
        f_close(&fil);
        return fres;
    }

    UINT bytesWrote;
    fres = f_write(&fil, data, strlen(data), &bytesWrote);
    if (fres != FR_OK)
    {
        myprintf("f_write error (%i)\r\n", fres);
        f_close(&fil);
        return fres;
    }

    // Write newline after GPS data
    const char newline[] = "\r\n";
    fres = f_write(&fil, newline, sizeof(newline) - 1, &bytesWrote);
    if (fres != FR_OK)
    {
        myprintf("f_write newline error (%i)\r\n", fres);
        f_close(&fil);
        return fres;
    }

    fres = f_close(&fil);
    if (fres != FR_OK)
    {
        myprintf("f_close error (%i)\r\n", fres);
    }

    return fres;
}

// System clock config, peripheral init and other HAL callbacks unchanged

void SystemClock_Config(void)
{
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

  __HAL_RCC_PWR_CLK_ENABLE();
  __HAL_PWR_VOLTAGESCALING_CONFIG(PWR_REGULATOR_VOLTAGE_SCALE3);

  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSI;
  RCC_OscInitStruct.HSIState = RCC_HSI_ON;
  RCC_OscInitStruct.HSICalibrationValue = RCC_HSICALIBRATION_DEFAULT;
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
  RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSI;
  RCC_OscInitStruct.PLL.PLLM = 16;
  RCC_OscInitStruct.PLL.PLLN = 336;
  RCC_OscInitStruct.PLL.PLLP = RCC_PLLP_DIV4;
  RCC_OscInitStruct.PLL.PLLQ = 2;
  RCC_OscInitStruct.PLL.PLLR = 2;
  if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK) { Error_Handler(); }

  RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK|RCC_CLOCKTYPE_SYSCLK
                              |RCC_CLOCKTYPE_PCLK1|RCC_CLOCKTYPE_PCLK2;
  RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
  RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
  RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV2;
  RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV1;

  if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_2) != HAL_OK) { Error_Handler(); }
}

static void MX_SPI1_Init(void)
{
	hspi1.Instance = SPI1;
  hspi1.Init.Mode = SPI_MODE_MASTER;
  hspi1.Init.Direction = SPI_DIRECTION_2LINES;
  hspi1.Init.DataSize = SPI_DATASIZE_8BIT;
  hspi1.Init.CLKPolarity = SPI_POLARITY_LOW;
  hspi1.Init.CLKPhase = SPI_PHASE_1EDGE;
  hspi1.Init.NSS = SPI_NSS_SOFT;
  hspi1.Init.BaudRatePrescaler = SPI_BAUDRATEPRESCALER_128;
  hspi1.Init.FirstBit = SPI_FIRSTBIT_MSB;
  hspi1.Init.TIMode = SPI_TIMODE_DISABLE;
  hspi1.Init.CRCCalculation = SPI_CRCCALCULATION_DISABLE;
  hspi1.Init.CRCPolynomial = 10;
  if (HAL_SPI_Init(&hspi1) != HAL_OK) { Error_Handler(); }
}

static void MX_USART2_UART_Init(void)
{
  huart2.Instance = USART2;
  huart2.Init.BaudRate = 115200;
  huart2.Init.WordLength = UART_WORDLENGTH_8B;
  huart2.Init.StopBits = UART_STOPBITS_1;
  huart2.Init.Parity = UART_PARITY_NONE;
  huart2.Init.Mode = UART_MODE_TX_RX;
  huart2.Init.HwFlowCtl = UART_HWCONTROL_NONE;
  huart2.Init.OverSampling = UART_OVERSAMPLING_16;
  if (HAL_UART_Init(&huart2) != HAL_OK) { Error_Handler(); }
}

static void MX_GPIO_Init(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  __HAL_RCC_GPIOC_CLK_ENABLE();
  __HAL_RCC_GPIOH_CLK_ENABLE();
  __HAL_RCC_GPIOA_CLK_ENABLE();
  __HAL_RCC_GPIOB_CLK_ENABLE();

  HAL_GPIO_WritePin(SD_CS_GPIO_Port, SD_CS_Pin, GPIO_PIN_RESET);

  GPIO_InitStruct.Pin = B1_Pin;
  GPIO_InitStruct.Mode = GPIO_MODE_IT_FALLING;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  HAL_GPIO_Init(B1_GPIO_Port, &GPIO_InitStruct);

  GPIO_InitStruct.Pin = SD_CS_Pin;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
  HAL_GPIO_Init(SD_CS_GPIO_Port, &GPIO_InitStruct);
}

void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
  if (htim->Instance == TIM1)
  {
    HAL_IncTick();
  }
}

void Error_Handler(void)
{
  __disable_irq();
  while (1) { }
}

#ifdef  USE_FULL_ASSERT
void assert_failed(uint8_t *file, uint32_t line)
{
  // User can add implementation here for assert failures
}
#endif
