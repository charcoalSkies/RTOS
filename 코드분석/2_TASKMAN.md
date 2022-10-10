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

</br>

### vTaskDelay 의 중요성

vTaskDelay를 통해 멀티태스킹을 구현 

vTaskDelay를 사용하지 않으면 선점중인 Task만 실행하고 다른 Task를 실행하지 못함



``` c
static void Task1( void const *pvParameters )
{
	const char *pcTaskName = "Task1";

	pvParameters = pvParameters; // for compiler warning

	/* Print out the name of this task. */
	printf( "%s is running\n", pcTaskName );

	printf("\n-------  Task1 information -------\n");
	printf("task1 name = %s \n",pcTaskGetName( xHandle1 ));
	printf("task1 priority = %d \n",(int)uxTaskPriorityGet( xHandle1 ));
//	printf("task1 status = %d \n",eTaskGetState( xHandle1 ));
	printf("----------------------------------\n");

	while(1) {
	/* TODO #3:
		코드를 실행 하여 보고
		vTaskDelay() 코드를 주석 처리한 후 그 결과를 설명한다 */
#if 1 // No comment
vTaskDelay (pdMS_TO_TICKS (1000)); // 1000ms 동안 blocked, 1000ms 후 ready (얘덕분에 멀티태스킹 가능) 블럭 상태 동안 다른 Task실행
// running은 비어있으면 안되므로 task 가 없다면(task 공백기가 생겼다면) Idle Task실행(우선순위 0번) vApplicationIdleHook 로 사용자가 작업 지정 가능

// 우선순위가 높다고 가장 오래 실행되는것이 아님
// 휴면을 적게 할수록 오래 실행

printf("a");
fflush(stdout);	// 문자 'a' 출력

// 디버깅시 주의!
// printf("hello,world!"); 개행문자를 안썼기 때문에 문자열 버퍼에 담김, 버퍼가 다 재워져야 화면에 출력
// fflush(stdout) => 표준 출력 stdout(printf)문자열을 강제로 배출하고 버퍼를 비운다
// fflush를 하게되면 UART 하드웨어 장치를 통해 문자 강제 출력(UART 매우 느림)
// fflush를 안넣으면 1초에 한번씩 출력을 안시킴(버퍼가 안찼기 때문에)

#endif // TODO #3

		task1timer++;
	}
}



// static void Task2 생략....




void vApplicationIdleHook (void)		// configUSE_IDLE_HOOK 1로 설정시 idle task 에서 지속적으로 호출되는 것을 볼 수 있음
{
	// 사용자가 작업 설정 가능함
	printf("."); fflush(stdout);
}
```

</br>
</br>
</br>
</br>


### 최종 정리

``` c
// 1. TASK 의 형태 
------------------------------------------------
// 유형 1 (무한루프 함수)
void YourTask(void *pvParameters)
{
	while(1) 
	{
		// do something
		vTaskDelay(1);	// 1ms 대기
		vTaskSuspend();
		vTaskPrioritySet();
		vTaskResume();
	}
}	// Task는 절대 return값이 있어서는 안됨


// 유형 2 (비무한루프 함수) 실행 후 스스로를 삭제
void YourTask(void *pvParameters)
{
	// do something
	vTaskDelete(NULL);	// 스스로 삭제
}	


// 태스크는 절대 리턴하면 안되므로 항상 'void'로 리턴형 사용 
//	ex) void userTask(void *pvParameters) { ... }





// 2. 태스크와 우선순위
------------------------------------------------
ㅁ 우선순위 갯수는 사실상 무제한 
ㅁ 큰 숫자가 높은 우선순위를 나타냄 
	ㅁ 낮은 우선순위 : 0(Idle Task), 1, 2, 3, ....
	ㅁ 높은 우선순위 : 100, 101, 102, ....

ㅁ 우선순위의 사용범위를 다음처럼 제한할 수 있음
#define configMAX_PRIORITIES	( 100 )	// 0~99까지 사용가능



ㅁ 동일한 우선순위도 사용 가능(Round Robin 지원) 
// 선점형 스케줄링의 하나로서, 프로세스들 사이에 우선순위를 두지 않고, 순서대로 시간단위(Time Quantum)로 CPU를 할당하는 방식


ㅁ 새로 생성된 태스크가 우선순위가 높다면 생성과 동시에 CPU 제어권을 할당받아 실행된다. 

ㅁ FreeRTOS 에서는 vTaskStartScheduler() 함수를 호출해서 멀티태스킹을 시작





// 3. TASK 생성
------------------------------------------------
// TASK CREATE
#include “FreeRTOS.h”
#include “task.h”
BaseType_t xTaskCreate( 
	TaskFunction_t pvTaskCode,
	const char * const pcName,
	unsigned short usStackDepth,
	void *pvParameters,
	UBaseType_t uxPriority,
	TaskHandle_t *pxCreatedTask );


//ex)
xTaskCreate(
	TASK1, 		   // TASK 함수	
	"TASK1", 	   // TASK 함수 설명 
	128, 		   // 128 word: stack size
	NULL, 		   // 태스크 전달 파라미터
	TASK_1_PRIO,   // 우선순위
	&xHandle1);	   // handle





// 4. vTaskSuspend
------------------------------------------------
vTaskSuspend( xHandle1 );

ㅁ 태스크를 일시정지 시킴 


// 5. vTaskResume
------------------------------------------------
vTaskSuspend( xHandle1 );

ㅁ 일시정지된 태스크를 다시 실행시킴

// ex)
void vAFunction( void )
{
	TaskHandle_t xHandle;
	/* Create a task, storing the handle to the created task in xHandle. */
	if( xTaskCreate( vTaskCode,
					"Demo task",
					STACK_SIZE,
					NULL,
					PRIORITY,
					&xHandle ) != pdPASS ) {
		/* The task was not created successfully. */
	}
	else {
		/* Use the handle to suspend the created task. */
		vTaskSuspend( xHandle );
		/* The suspended task will not run during this period, unless another task calls vTaskResume( xHandle ). */
		/* Resume the suspended task again. */
		vTaskResume( xHandle );
		/* The created task is again available to the scheduler and can enter The Running state. */
}
}




// 6. vTaskPrioritySet
------------------------------------------------
// TASK_1_PRIO(기존) -> TASK_3_PRIO
// xHandle1에 할당된 TASK의 우선순위를 TASK_3_PRIO로 변경
vTaskPrioritySet(xHandle1, TASK_3_PRIO); 

// ex)
xTaskCreate( (TaskFunction_t)Task2,
					"Task2",
					256,
					(void*)Param,
					TASK_2_PRIO,
					&xHandle2 );

vTaskSuspend(xHandle1); // ready -> blocked
vTaskPrioritySet(xHandle1, TASK_3_PRIO); // TASK_1_PRIO -> TASK_3_PRIO
vTaskResume(xHandle1);	// blocked -> ready



// 7. vTaskDelay
------------------------------------------------
// vTaskDelay 덕분에 멀티태스킹 가능
// vTaskDelay를 사용하지 않으면 선점중인 태스크만 실행하고 다른 태스크를 실행할 수 없음
vTaskDelay (pdMS_TO_TICKS (1)); 
// 1ms 동안 blocked, 1ms 후 ready  블럭 상태 동안 다른 Task실행
// running은 비어있으면 안되므로 task 가 없다면(task 공백기가 생겼다면) Idle Task실행(우선순위 0번)

// vApplicationIdleHook 함수를 사용해서 Idle Task를 사용자가 정의할 수 있음 

// 우선순위가 높다고 가장 오래 실행되는것이 아님
// 휴면을 적게 할수록 오래 실행



// 8. vTaskDelete
------------------------------------------------
/* delete self task */
vTaskDelete (xHandleMain);

// xHandleMain에 할당된 TASK를 삭제


// ex)
void vAnotherFunction( void )
{
	TaskHandle_t xHandle;
	/* Create a task, storing the handle to the created task in xHandle. */
	if( xTaskCreate( vTaskCode,
					"Demo task",
					STACK_SIZE,
					NULL,
					PRIORITY,
					&xHandle ) /* The address of xHandle is passed in as the last parameter to xTaskCreate() to obtain a handle to the task being created. */
			!= pdPASS ) {
				/* The task could not be created because there was not enough FreeRTOS heap memory available for the task data structures and stack to be allocated. */
	}
	else
	{
		/* Delete the task just created. Use the handle passed out of xTaskCreate() to reference the subject task. */
		vTaskDelete( xHandle );
	}
	/* Delete the task that called this function by passing NULL in as the vTaskDelete() parameter. The same task (this task) could also be deleted by passing in a valid handle to itself. */
	vTaskDelete( NULL );
}
```