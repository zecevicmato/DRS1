```c
#include <stdio.h>
#include "xparameters.h"
#include "xtmrctr.h"
#include "xintc.h"
#include "xgpio.h"


//definiranje konstante za redni broj timera koji se koristi (unutar timer komponente postoje dva timera)
#define TIMER_CNTR_0 0

//***********************************************TO DO 1************************************************//
//*****************************Postaviti vrijednost konstante RESET_VALUE*******************************//
//****************************************t = (CONST + 2) x T*******************************************//
//*********************frekvenciju pogledati u komponenti clock_generator u XPS-u***********************//
#define RESET_VALUE 49999998

//******************************************************************************************************//

/********************* Prototipi funkcija *************************/
int TmrCtrIntrInit(XIntc* IntcInstancePtr,
 XTmrCtr* TmrInstancePtr,
 u16 TimerDeviceId,
 u16 IntcDeviceId,
 u16 IntrSourceId,
 u8 TmrCtrNumber);

void TimerCounterHandler(void *CallBackRef, u8 TmrCtrNumber);

//**************Deklaracija varijable InterruptControler (XIntc) i TimerCounterInst (XTmrCtr)***********//
XIntc InterruptController;
XTmrCtr TimerCounterInst;
int BrojacHocovihPojena = 2;

int main(void)
{
	print("-- Start of the program! --\r\n");

	XGpio leds;
	XGpio_Initialize(&leds, XPAR_LEDS_8BITS_DEVICE_ID);

	int Status;

	//Poziv funkcije za inicijalizaciju timera i upravljača prekidima
	Status = TmrCtrIntrInit(&InterruptController,
							&TimerCounterInst,
							XPAR_DELAY_DEVICE_ID,
							XPAR_INTC_0_DEVICE_ID,
							XPAR_INTC_0_TMRCTR_0_VEC_ID,
							TIMER_CNTR_0);

	if (Status != XST_SUCCESS){
		return XST_FAILURE;
	}

	//***********************************************TO DO 2***********************************************//
	//******************************************Pokrenuti timer - XTmrCtr_Start(...)***********************//
	XTmrCtr_Start(&TimerCounterInst, 0);

	//*****************************************************************************************************//

	while (1)
	{
		if (BrojacHocovihPojena == 10)
		{
			BrojacHocovihPojena = 2;
		}
			XGpio_DiscreteWrite(&leds, 1, BrojacHocovihPojena);
	}

	print("-- End of the program! --\r\n");
	return XST_SUCCESS;
}

/********************************************************************/
/**
* Inicijalizacija timera i upravljača prekidima.
* Funkcija prima sljedeće parametre:
*
* @paramIntcInstancePtr - pokazivač na varijablu tipa XIntc,
* @paramTmrCtrInstancePtr - pokazivač na varijablu tipa XTmrCtr,
* @paramTimerDeviceId - vrijednost konstante XPAR_<TmrCtr_instance>_DEVICE_ID iz datoteke xparameters.h,
* @paramIntcDeviceId - vrijednost konstante XPAR_<Intc_instance>_DEVICE_ID iz datoteke xparameters.h,
* @paramIntrSourceId - vrijednost konstante XPAR_<INTC_instance>_<TmrCtr_instance>_INTERRUPT_INTR iz datoteke xparameters.h,
* @paramTmrCtrNumber - redni broj timera koji se inicijalizira.
*
* @returnXST_SUCCESS ako je inicijalizacija uspješna, a u suprotno funkcija vraća XST_FAILURE
*
*********************************************************************/
int TmrCtrIntrInit(XIntc* IntcInstancePtr,
 XTmrCtr* TmrCtrInstancePtr,
 u16 TimerDeviceId,
 u16 IntcDeviceId,
 u16 IntrSourceId,
 u8 TmrCtrNumber)
{
	int Status;

	print("Init STARTED\r\n");

	//***********************************************TO DO 3************************************************//
	//*************************Inicijalizirati timer - XTmrCtr_Initialize(...)******************************//
	Status = XTmrCtr_Initialize(&TimerCounterInst, 0);

	//*****************************************************************************************************//
	if (Status != XST_SUCCESS) {
		print("Timer Initialize FAILED\r\n");
		return XST_FAILURE;
	}
	print("Timer Initialize SUCCESS\r\n");

	//**********************************************TO DO 4*************************************************//
	//*******************Inicijalizirati upravljač prekidima - XIntc_Initialize(...)************************//

	Status = XIntc_Initialize(&InterruptController, XPAR_XPS_INTC_0_DEVICE_ID);
	//*****************************************************************************************************//
	if (Status != XST_SUCCESS) {
		print("Intc Initialize FAILED\r\n");
		return XST_FAILURE;
	}
	print("Intc Initialize SUCCESS\r\n");

	/*
	 * Povezivanje upravljača prekida s rukovateljem prekida koji se
	 * poziva kada se dogodi prekid. Rukovatelj prekida obavlja
	 * specifične zadatke vezane za rukovanje prekidima.
	 */
	Status = XIntc_Connect(IntcInstancePtr, IntrSourceId,
	 (XInterruptHandler)XTmrCtr_InterruptHandler,
	 (void *)TmrCtrInstancePtr);

	if (Status != XST_SUCCESS) {
		print("Intc Connect FAILED\r\n");
		return XST_FAILURE;
	}
	print("Intc Connect SUCCESS\r\n");

	//***********************************************TO DO 5***********************************************//
	//*****************Postaviti mod rada upravljača prekida na REAL MODE - XIntc_Start(...)***************//

	XIntc_Start(&InterruptController, XIN_REAL_MODE);
	//*****************************************************************************************************//
	if (Status != XST_SUCCESS) {
		print("Intc Start FAILED\r\n");
		return XST_FAILURE;
	}
	print("Intc Start SUCCESS\r\n");

	//**********************************************TO DO 6******************************+****************//
	//**************************Omogućiti rad upravljača prekidima - XIntc_Enable(...)*********************//

	XIntc_Enable(&InterruptController, XPAR_XPS_INTC_0_DEVICE_ID);
	//*****************************************************************************************************//

	//Omogućavanje microblaze prekida.
	microblaze_enable_interrupts();

	/*
	 * Postavljanje prekidne rutine koja će biti pozvana kada se dogodi prekid
	 * od strane timera. Kao parametri predaju se pokazivač na komponentu za
	 * koju se postavlja prekidna rutina, naziv prekidne rutine (funkcije)
	 * i pointer na timer, kako bi prekidna rutina mogla pristupiti timeru.
	 */
	XTmrCtr_SetHandler(TmrCtrInstancePtr,
	 TimerCounterHandler,
	 TmrCtrInstancePtr);

	//**********************************************TO DO 7******************************+****************//
	//*************************Postaviti postavke timera - XTmrCtr_SetOptions(...)*************************//
	//************Omogućiti prekide, omogućiti auto reload, odabrati brojanje prema dolje******************//
	XTmrCtr_SetOptions(&TimerCounterInst, 0, XTC_DOWN_COUNT_OPTION	|	XTC_AUTO_RELOAD_OPTION	|	XTC_INT_MODE_OPTION);


	//*****************************************************************************************************//

	//**********************************************TO DO 8******************************+****************//
	//******************Postaviti početnu vrijednost timera - XTmrCtr_SetResetValue(...)******************//
	XTmrCtr_SetResetValue(&TimerCounterInst, 0, RESET_VALUE);

	//*****************************************************************************************************//

	print("Init FINISHED\r\n");
	return XST_SUCCESS;
}

/*
 * Prekidna rutina koja se poziva kada timer generira prekid.
 * Funkcija prima pokazivač na void parametar CallBackRef
 * koji se cast-a na pokazivač tipa XTmrCtr.
 * Ovaj parametar je napravljen kako bi se pokazao način na
 * koji se unutar prekidne rutine može pristupiti timer
 * komponenti i njenim funkcijama.
*/
void TimerCounterHandler(void *CallBackRef, u8 TmrCtrNumber)
{
	print("Interrupt Handler!\r\n");

	XTmrCtr *InstancePtr = (XTmrCtr *)CallBackRef;


	BrojacHocovihPojena++;
}

```
