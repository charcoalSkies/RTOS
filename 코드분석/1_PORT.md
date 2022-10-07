# 1. _PORT 주요 코드 해석

</br>
</br>
</br>

### printf() 함수 사용 하기 위한 __io_putchar overriding
</br>

``` c
int __io_putchar(int ch)
{
 if ( ch == '\n' )
	 HAL_UART_Transmit(&huart2, (uint8_t*)&"\r", 1, HAL_MAX_DELAY);
 HAL_UART_Transmit(&huart2, (uint8_t*)&ch, 1, HAL_MAX_DELAY);
 return ch;
}
```

</br>
</br>
</br>

### osKernelStart() 함수 전 Task 생성

main함수 내부에서 osKernelStart() 함수 전에 Task를 생성해야 한다. 

``` c
  MX_FREERTOS_Init();

  /* Start scheduler */
  osKernelStart();
```

</br>

### osThreadDef() 함수를 통한 Task 생성 (CMSIS_OS 사용)

``` c
/* Create the thread(s) */
  /* definition and creation of defaultTask */
  osThreadDef(defaultTask, StartDefaultTask, osPriorityNormal, 0, 128);
  defaultTaskHandle = osThreadCreate(osThread(defaultTask), NULL); 
```

</br>

### osThreadDef() 함수의 인자

``` c
void StartDefaultTask(void const * argument)
{
  /* USER CODE BEGIN StartDefaultTask */
	printf("[TASK]StartDefaultTask\n");
  /* Infinite loop */
  for(;;)
  {
	  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_RESET);
	  osDelay(500);
	  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET);
	  osDelay(500);
  }
  /* USER CODE END StartDefaultTask */
}
```

