/*
 * This file is part of the µOS++ distribution.
 *   (https://github.com/micro-os-plus)
 * Copyright (c) 2014 Liviu Ionescu.
 *
 * Permission is hereby granted, free of charge, to any person
 * obtaining a copy of this software and associated documentation
 * files (the "Software"), to deal in the Software without
 * restriction, including without limitation the rights to use,
 * copy, modify, merge, publish, distribute, sublicense, and/or
 * sell copies of the Software, and to permit persons to whom
 * the Software is furnished to do so, subject to the following
 * conditions:
 *
 * The above copyright notice and this permission notice shall be
 * included in all copies or substantial portions of the Software.
 *
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
 * EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES
 * OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
 * NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT
 * HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY,
 * WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
 * FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR
 * OTHER DEALINGS IN THE SOFTWARE.
 */

// ----------------------------------------------------------------------------

#include <stdio.h>
#include <stdlib.h>
#include "diag/trace.h"

/* Kernel includes. */
#include "FreeRTOS.h"
#include "task.h"
#include "queue.h"
#include "timers.h"
#include "semphr.h"

#define CCM_RAM __attribute__((section(".ccmram")))

static void SendAutoReloadTimerCallback();
static void ReceiveAutoReloadTimerCallback();
void Init ();


/*-----------------------------------------------------------*/
// ----------------------------------------------------------------------------
//
// Semihosting STM32F4 empty sample (trace via DEBUG).
//
// Trace support is enabled by adding the TRACE macro definition.
// By default the trace messages are forwarded to the DEBUG output,
// but can be rerouted to any device or completely suppressed, by
// changing the definitions required in system/src/diag/trace-impl.c
// (currently OS_USE_TRACE_ITM, OS_USE_TRACE_SEMIHOSTING_DEBUG/_STDOUT).
//

// ----- main() ---------------------------------------------------------------

// Sample pragmas to cope with warnings. Please note the related line at
// the end of this function, used to pop the compiler diagnostics status.
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wunused-parameter"
#pragma GCC diagnostic ignored "-Wmissing-declarations"
#pragma GCC diagnostic ignored "-Wreturn-type"


//define handlers of tasks,timers,queue and semaphores
TaskHandle_t SenderTaskHandle=NULL;
TaskHandle_t ReceiverTaskHandle=NULL;
SemaphoreHandle_t SendSemaphore;
SemaphoreHandle_t ReceiveSemaphore;
TimerHandle_t SendAutoReloadTimer;
TimerHandle_t ReceiveAutoReloadTimer;
QueueHandle_t myQueue;

//array of different periods of sender timer and a pointer pointing to the first location before it
int Tsender[6]={100, 140, 180, 220, 260, 300};
int * current_sender_period=Tsender-1;


int SucessSend,BlockedMessages,MessagesNumber=0;
//defining sender task
void SenderTask (void *p){
	char sendertxt[50];
	BaseType_t Status;
	while(1){
		if(xSemaphoreTake(SendSemaphore,portMAX_DELAY)){
		sprintf(sendertxt,"Time is %lu",xTaskGetTickCount());
		Status=xQueueSend(myQueue,&sendertxt,0);
		if(Status==pdPASS) {SucessSend++;} else {BlockedMessages++;}
		MessagesNumber++;
		}
	}
}


int ReceivedMessages=0;
void ReceiverTask (void *p){
	char Receivertxt[50];
	while(1){
		if(xSemaphoreTake(ReceiveSemaphore,portMAX_DELAY)){
		if(myQueue!=0){
			if(xQueueReceive(myQueue,&Receivertxt,0)){
			//	printf("%s\r\n",Receivertxt);
				ReceivedMessages++;
			}
		}
		if (ReceivedMessages==500) {Init();}
	}
	}
}

int EndFlag=0; //flag indicates reaching the last value of the array of sender timer periods
int first_time=1;// a flag indicates the first call of Init not to change timer period in this call
void Init (){
	if(first_time){
		//defining period of receiver timer and initial period of sender timer
		//using pdMS_TO_TICKS() to convert mSec to ticks
		#define RECEIVE_TIMER_PERIOD pdMS_TO_TICKS( 200 )
		#define SEND_TIMER_PERIOD pdMS_TO_TICKS(100)
	}
	if(!first_time){
	printf("when sender period= %dmSec :\n",*current_sender_period)	;
	printf("total number of successfully sent messages=%d\n",SucessSend);
	printf("total number of blocked message=%d\n",BlockedMessages);}
	if(EndFlag!=1){
	if(!first_time){SucessSend=0; BlockedMessages=0; ReceivedMessages=0;}
	xQueueReset( myQueue );
	current_sender_period++;//pointing to the next sender timer period
	if(!first_time)xTimerChangePeriod( SendAutoReloadTimer,*current_sender_period,0);

	first_time=0;
	if ((*current_sender_period)==300){EndFlag=1;}
	}
	else {
		//destroying timers and printing "Game Over"
		xTimerDelete( SendAutoReloadTimer,0 );
		xTimerDelete( ReceiveAutoReloadTimer,0);
		printf("Game Over\n");
		exit(1);
	}
}

int
main(int argc, char* argv[])
{
	//creating semaphores
	SendSemaphore=xSemaphoreCreateBinary();
	ReceiveSemaphore=xSemaphoreCreateBinary();
	//creating queue and defining its size
	int queueSize=2;
	myQueue=xQueueCreate(queueSize,sizeof(char[50]));

	Init();//calling Init for the first time

	//creating periodic timers
	SendAutoReloadTimer=xTimerCreate("SendTimer",SEND_TIMER_PERIOD ,pdTRUE,0,SendAutoReloadTimerCallback);
	ReceiveAutoReloadTimer=xTimerCreate("ReceiveTimer",RECEIVE_TIMER_PERIOD,pdTRUE,0,ReceiveAutoReloadTimerCallback);
	xTimerStart( SendAutoReloadTimer, 0 );
	xTimerStart( ReceiveAutoReloadTimer, 0 );


	//creating sender and receiver tasks
	//receiver task is of higher priority than sender task
	xTaskCreate(SenderTask,"send",1000,NULL,1,NULL);
	xTaskCreate(ReceiverTask,"Receive",1000,NULL,2,NULL);


	vTaskStartScheduler();
	return 0;
}

#pragma GCC diagnostic pop

// ----------------------------------------------------------------------------

static void SendAutoReloadTimerCallback()
{
	xSemaphoreGive(SendSemaphore);
}

static void ReceiveAutoReloadTimerCallback()
{

	xSemaphoreGive(ReceiveSemaphore);
}


void vApplicationMallocFailedHook( void )
{
	/* Called if a call to pvPortMalloc() fails because there is insufficient
	free memory available in the FreeRTOS heap.  pvPortMalloc() is called
	internally by FreeRTOS API functions that create tasks, queues, software
	timers, and semaphores.  The size of the FreeRTOS heap is set by the
	configTOTAL_HEAP_SIZE configuration constant in FreeRTOSConfig.h. */
	for( ;; );
}
/*-----------------------------------------------------------*/

void vApplicationStackOverflowHook( TaskHandle_t pxTask, char *pcTaskName )
{
	( void ) pcTaskName;
	( void ) pxTask;

	/* Run time stack overflow checking is performed if
	configconfigCHECK_FOR_STACK_OVERFLOW is defined to 1 or 2.  This hook
	function is called if a stack overflow is detected. */
	for( ;; );
}
/*-----------------------------------------------------------*/

void vApplicationIdleHook( void )
{
volatile size_t xFreeStackSpace;

	/* This function is called on each cycle of the idle task.  In this case it
	does nothing useful, other than report the amout of FreeRTOS heap that
	remains unallocated. */
	xFreeStackSpace = xPortGetFreeHeapSize();

	if( xFreeStackSpace > 100 )
	{
		/* By now, the kernel has allocated everything it is going to, so
		if there is a lot of heap remaining unallocated then
		the value of configTOTAL_HEAP_SIZE in FreeRTOSConfig.h can be
		reduced accordingly. */
	}
}

void vApplicationTickHook(void) {
}

StaticTask_t xIdleTaskTCB CCM_RAM;
StackType_t uxIdleTaskStack[configMINIMAL_STACK_SIZE] CCM_RAM;

void vApplicationGetIdleTaskMemory(StaticTask_t **ppxIdleTaskTCBBuffer, StackType_t **ppxIdleTaskStackBuffer, uint32_t *pulIdleTaskStackSize) {
  /* Pass out a pointer to the StaticTask_t structure in which the Idle task's
  state will be stored. */
  *ppxIdleTaskTCBBuffer = &xIdleTaskTCB;

  /* Pass out the array that will be used as the Idle task's stack. */
  *ppxIdleTaskStackBuffer = uxIdleTaskStack;

  /* Pass out the size of the array pointed to by *ppxIdleTaskStackBuffer.
  Note that, as the array is necessarily of type StackType_t,
  configMINIMAL_STACK_SIZE is specified in words, not bytes. */
  *pulIdleTaskStackSize = configMINIMAL_STACK_SIZE;
}

static StaticTask_t xTimerTaskTCB CCM_RAM;
static StackType_t uxTimerTaskStack[configTIMER_TASK_STACK_DEPTH] CCM_RAM;

/* configUSE_STATIC_ALLOCATION and configUSE_TIMERS are both set to 1, so the
application must provide an implementation of vApplicationGetTimerTaskMemory()
to provide the memory that is used by the Timer service task. */
void vApplicationGetTimerTaskMemory(StaticTask_t **ppxTimerTaskTCBBuffer, StackType_t **ppxTimerTaskStackBuffer, uint32_t *pulTimerTaskStackSize) {
  *ppxTimerTaskTCBBuffer = &xTimerTaskTCB;
  *ppxTimerTaskStackBuffer = uxTimerTaskStack;
  *pulTimerTaskStackSize = configTIMER_TASK_STACK_DEPTH;
}

