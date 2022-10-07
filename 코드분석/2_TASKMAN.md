# 1. _TASKMAN 주요 코드 해석

</br>
</br>
</br>

### 우선순위 및 핸들러명 셋팅 

``` c
/* task's priority */
#define TASK_MAIN_PRIO	20
#define TASK_1_PRIO		10
#define TASK_2_PRIO		 9
#define TASK_3_PRIO		 8


TaskHandle_t xHandleMain, xHandle1, xHandle2;

// 각 Task1과 Task2는 내부에 무한루프 구문 존재
```

### USER_THREADS 내부에 TASK 생성 

``` c
  /* USER CODE BEGIN RTOS_THREADS */
  USER_THREADS();
  /* USER CODE END RTOS_THREADS */

  /* Start scheduler */
  osKernelStart();
```

</br>

### TaskMain 을 우선 순위가 가장 높은 TASK로 생성한다.

``` c
void USER_THREADS( void )
{
	/* Setup the hardware for use with the Beagleboard. */
	//prvSetupHardware();
#ifdef CMSIS_OS
	osThreadDef(defaultTask, TaskMain, osPriorityHigh, 0, 256);
	defaultTaskHandle = osThreadCreate(osThread(defaultTask), NULL);
#else
	/* Create one of the two tasks. */
	xTaskCreate(	(TaskFunction_t)TaskMain,		/* Pointer to the function that implements the task. */
					"TaskMain",	/* Text name for the task.  This is to facilitate debugging only. */
					128,		// 워드 단위
					NULL,		/* We are not using the task parameter. */
					TASK_MAIN_PRIO,	/* This task will run at this priority */
					&xHandleMain );		// 위에서 설정한 TaskHandle_t 로 선언한 변수를 XTaskCreate를 통해 할당하게 되면 삭제 등 제어가 가능해진다 
#endif
}
```

</br>

### TaskMain 내부에서 Task1을 생성하고 vTaskDelete() 함수를 통해 자신을 삭제한다. 
### 따라서 현재 등록되어 있는 task 는 Task1 뿐
``` c
static void TaskMain( void const *pvParameters )
{
	const char *pcTaskName = "TaskMain";
	struct Param_types *Param;

	pvParameters = pvParameters; // for compiler warning

	/* Print out the name of this task. */
	printf( "%s is running\r\n", pcTaskName );

	// TASK CREATE
	/* TODO #1:
		Task1을 생성
		use 'xTaskCreate' */
#if 1
	xTaskCreate(	(TaskFunction_t)Task1,		/* Pointer to the function that implements the task. */
						"Task1",	/* Text name for the task.  This is to facilitate debugging only. */
						128,		// 워드 단위 
						NULL,		/* We are not using the task parameter. */
						TASK_1_PRIO,	// 우선순위 (숫자가 클수록 높은 우선순위)
						&xHandle1 );    // 위에서 설정한 TaskHandle_t 로 선언한 변수를 XTaskCreate를 통해 할당하게 되면 삭제 등 제어가 가능해진다 
#endif // TODO #1

	/* Create the other task in exactly the same way. */
	Param = &Param_Tbl;		/* get parameter tbl addr */
	Param->P1 = 111111;		/* set parameter */
	Param->P2 = 222222;

	/* delete self task */
	vTaskDelete (xHandleMain);	// vTaskDelete (NULL);
}
```

</br>

### Task1 생성(우선순위 2순위로 대기) -> Task2 생성(우선순위 3순위로 대기) -> 

### Task1(vTaskSuspend) -> Task2 (우선순위 2순위로 대기) -> MainTask 삭제 (우선순위 1순위 제거)

### 결과 : Task2 실행

``` C
static void TaskMain( void const *pvParameters )
{
	const char *pcTaskName = "TaskMain";
	struct Param_types *Param;

	pvParameters = pvParameters; // for compiler warning

	/* Print out the name of this task. */
	printf( "%s is running\r\n", pcTaskName );

	// TASK CREATE
	/* TODO #1:
		Task1을 생성
		use 'xTaskCreate' */
#if 1
	xTaskCreate(	(TaskFunction_t)Task1,		/* Pointer to the function that implements the task. */
						"Task1",	/* Text name for the task.  This is to facilitate debugging only. */
						128,		// 워드 단위임 /
						NULL,		/* We are not using the task parameter. */
						TASK_1_PRIO,	/* This task will run at this priority */
						&xHandle1 );	// 위에서 설정한 TaskHandle_t 로 선언한 변수를 XTaskCreate를 통해 할당하게 되면 삭제 등 제어가 가능해진다 
#endif // TODO #1

	/* Create the other task in exactly the same way. */
	Param = &Param_Tbl;		/* get parameter tbl addr */
	Param->P1 = 111111;		/* set parameter */
	Param->P2 = 222222;
#ifdef CMSIS_OS
	osThreadDef(Task2, (void const *)Task2, osPriorityBelowNormal, 0, 256);
	xHandle2 = osThreadCreate (osThread(Task2), (void*)Param);
#else
	xTaskCreate( (TaskFunction_t)Task2,
					"Task2",        
					256,            // 공간 할당(워드)
					(void*)Param,   // Task2 함수의 인자로 전달
					TASK_2_PRIO,    // 우선순위가 현재 가장 낮음
					&xHandle2 );    // 위에서 설정한 TaskHandle_t 로 선언한 변수를 XTaskCreate를 통해 할당하게 되면 삭제 등 제어가 가능해진다
#endif

	/* TODO #2:
		Task1을 중지
		use 'vTaskSuspend' */
#if 1
	vTaskSuspend(xHandle1);

#endif // TODO #2

	/* delete self task */
	vTaskDelete (xHandleMain);	// vTaskDelete (NULL);
}

```

