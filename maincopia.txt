#include <p18f4553.h>
#include <stdio.h>
#include <stdlib.h> 
#include "interface.h"
#include "command.h"
#include "AtWareGraphics.h"
#include "teclado.h"

//////////////////////////////////
//			Graficos			//
//////////////////////////////////
/*
Esta función es invocada por todas las funciones de salida estándar <stdio.h>
*/
int _user_putc(char c)
{
	//Realizar la escritura del caracter proveniente de STDOUT mediante la biblioteca gráfica.
	//El cursor de caracteres GLCD_Column y GLCD_Line puede cambiarse en cualquier momento a manera de gotoxy antes de escribir
	return ATGConsolePutChar(c);
}

/*
 ----------- GRAPHICS LCD HARDWARE ABSTRACTION LAYER --------------------
 By Cornejo.
 
 Este es un ejemplo de HAL funcionando en mi configuración local. MPU 48MHz, GLCD, a máxima velocidad estable.
 
 Indico pues, será necesario ver los sincrogramas y los tiempos necesarios que deberán cumplirse para 
 que la comunicación sea estable. Factores como la longitud de los cables y las capacitancias parásitas
 pueden aumentar los requerimientos de temporización. Por favor, experimenten. Traten con 6 nops, y vayan reduciendo
 hasta lograr un mínimo estable. Esto debido a que los chips son ensamblados por diferentes fabricantes de módulos.
 
 No usar ESS_DelayMS dentro de la implementación de una HAL un milisegundo es una eternidad.... Las operaciones gráficas son de alta frecuencia y 
 computacionalmente costosos. A menos que quieran ver como se escribe pixel por pixel.
 
*/

//BUS DE CONTROL (PINOUT DEFINITION)
#define GLCD_CS1 PORTBbits.RB7   
#define GLCD_CS2 PORTBbits.RB6
#define GLCD_DI  PORTBbits.RB5
#define GLCD_RW  PORTBbits.RB4
#define GLCD_E	 PORTBbits.RB3
#define GLCD_TRIS_CS1 TRISBbits.TRISB7
#define GLCD_TRIS_CS2 TRISBbits.TRISB6
#define GLCD_TRIS_DI  TRISBbits.TRISB5
#define GLCD_TRIS_RW  TRISBbits.TRISB4
#define GLCD_TRIS_E	  TRISBbits.TRISB3
#define GLCD_TRIS_OUTPUT_MASK 0x07      //Bits 7:3 del puerto B como salidas
//BUS DE DATOS (PINOUT DEFINITION)
#define GLCD_TRIS_DATA TRISD
#define GLCD_PORT_DATA PORTD


//HAL de lectura de datos del GLCD
void User_ATGReadData(void)
{
	do {User_ATGReadStatus();} while(g_ucGLCDDataIn&GLCD_STATUSREAD_BUSY);
	GLCD_RW=1;
	GLCD_DI=1;
	GLCD_E=1;
	Nop();
	GLCD_E=0;
	do {User_ATGReadStatus();} while(g_ucGLCDDataIn&GLCD_STATUSREAD_BUSY);
	GLCD_RW=1;
	GLCD_DI=1;
	GLCD_E=1;
	Nop();
	Nop();
	g_ucGLCDDataIn=GLCD_PORT_DATA;
	GLCD_E=0;
	GLCD_CS1=1;
	GLCD_CS2=1;
}
//HAL  de escritura de datos al GLCD
void  User_ATGWriteData(void)
{
	do {User_ATGReadStatus();} while(g_ucGLCDDataIn&GLCD_STATUSREAD_BUSY);
	GLCD_PORT_DATA=g_ucGLCDDataOut;
	GLCD_TRIS_DATA=0x00;
	GLCD_RW=0;
	GLCD_DI=1;
	GLCD_E=1;
	Nop();
	GLCD_E=0;
	GLCD_CS1=1;
	GLCD_CS2=1;
}
//HAL de escritura de comandos al GLCD
void  User_ATGWriteCommand(void)
{
	do {User_ATGReadStatus();} while(g_ucGLCDDataIn&GLCD_STATUSREAD_BUSY);
	GLCD_PORT_DATA=g_ucGLCDDataOut;
	GLCD_TRIS_DATA=0x00;
	GLCD_RW=0;
	GLCD_DI=0;
	GLCD_E=1;
	Nop();
	GLCD_E=0;
	GLCD_CS1=1;
	GLCD_CS2=1;
}
//HAL de lectura del bit de BUSY del GLCD(g_ulGLCDDevChild)
void User_ATGReadStatus(void)
{
	GLCD_TRIS_DATA=0xff;
	if(g_ucGLCDDevChild)
		GLCD_CS1=0;
	else 
		GLCD_CS2=0;
	GLCD_RW=1;
	GLCD_DI=0;
	GLCD_E=1;
	Nop();
	Nop();
	g_ucGLCDDataIn=GLCD_PORT_DATA;
	GLCD_E=0;
}

//////////////////////////////////////
//			Baumanometro			//
//////////////////////////////////////

#define BOMBA 	 	PORTCbits.RC0   
#define VALVULA  	PORTCbits.RC1
#define START	 	PORTCbits.RC7
#define MICROFONO   PORTCbits.RC6

	//Constantes
	const float fVs   	= 5;
	const float fError 	= 0.028;

	//Variables
	float fVout 	= 0;
	float fBits 	= 0;
	float fPresion_kPa	= 0;
	float fPresion_mmHg	= 0;


void SensarPresion();

void SensarPresion()
{	
	ADCON0bits.ADON = 1;			//Habilita el ADC
	ESS_Delay(1);
	ADCON0bits.GO	= 1;			//Inicia conversión
	ESS_Delay(1);	
	
	fBits = (float) ADRES;
	fVout = 0.00012060546875 * ADRES;	
	fPresion_kPa = (((fVout - fError) / fVs) -0.04)* 777.7259293824856;
	fPresion_mmHg= fPresion_kPa * 7.500616827042;
	
	
	//printf("%d bits\n", (int) fBits);
	//printf("%d mV\n", (int) (fVout * 1000));
	//printf("%d kPa\n", (int) fPresion_kPa);
	//printf("%d mmHg\n", (int) fPresion_mmHg);
	
	ADCON0bits.ADON	= 0;			//Deshabilita ADC
	ADCON0bits.GO	= 0;
}	

void main(void)
{	
	int nPULxmin 	= 62;
	float fSYS 	 	= 0;
	float fDIA		= 0;
	int nStart  	= 0;
	int nEnd	 	= 0;
	int nSecuencia 	= 0;

	int fPresionAnterior = 0;
	int nContador = 0;

	int nImprimir = 0;
	
	//////////////////////////////////
	//			Graficos			//
	//////////////////////////////////
											
	//Se le indica la salida estándar que debe usar la función definida por el usuario para imprimir un caracter
	stdout=_H_USER;

	//Bus de control GLCD (y otros periféricos) como salida 
	PORTB=0xc0;    
	TRISB=GLCD_TRIS_OUTPUT_MASK; 
	
	//Inicializar el Modulo GLCD (Limpieza del estado interno del GLCD tras el reset o power on)
	ATGInit();
	//Activar la polarización del cristal líquido. 
	ATGShow();
	
	//Establecer las tintas de brocha (relleno) y pluma (filete)
	g_GLCDBrush=GLCD_BLACK;
	g_GLCDPen=GLCD_WHITE;	
	
	//////////////////////////////////////////////////
	//		Sensores y actuadores Baumanometro		//
	//////////////////////////////////////////////////
		
	PORTC = 0x00;
	TRISC = 0xf0;
	
	//////////////////////////////////
	//				ADC				//
	//////////////////////////////////
	
	ADCON0 = 0x00; 				//Habilita el modulo A/D
	ADCON1 = 0x1D;				//Configura RA0, RA1 como Analogicos		
			
	while(1)
	{	
		VALVULA = 0;
		BOMBA = 0;
		if(!START)
		{
			//Limpiar el GLCD con el color de brocha
			ATGClear();
				
			//Indica donde se inicia a dibujar
			g_Column = 0;
			g_Line	 = 3;
			
			printf("Iniciando medicion...");
			ESS_Delay(2000);

			while(nSecuencia!=3)
			{
				if(nSecuencia == 0)
				{
					BOMBA = 1;
					nSecuencia = 1;
					nImprimir = 0;
				}
				if(nSecuencia == 1 )
				{
					SensarPresion();

					if(fPresion_mmHg > 110)
					{
						if(fPresion_mmHg >= fPresionAnterior - 2 || fPresion_mmHg <= fPresionAnterior + 2)
						{
							nContador++;
							fPresionAnterior = fPresion_mmHg;
						}
						else
						{
							nContador = 0;
							fPresionAnterior = fPresion_mmHg;
						}
	
						if (nContador == 100)
						{
							fSYS = fPresion_mmHg;
							BOMBA = 0;
							ESS_Delay(30);
							VALVULA = 1;
							nSecuencia = 2;
							nContador = 0;
						}
					}
					
					nImprimir++;

					if(nImprimir == 10)	
					{
						//Limpiar el GLCD con el color de brocha
						ATGClear();
					
						//Indica donde se inicia a dibujar
						g_Column = 0;
						g_Line	 = 1;
						
						printf("Midiendo...\n");
						/*printf("%d bits\n", (int) fBits);
						printf("%d mV\n", (int) (fVout * 1000));
						printf("%d kPa\n", (int) fPresion_kPa);
						printf("%d mmHg\n", (int) fPresion_mmHg);*/
						nImprimir = 0;
					}	
					ESS_Delay(10);
				}
				if(nSecuencia == 2)
				{		
					SensarPresion();

					if(fPresion_mmHg < 85)
					{
						if(fPresion_mmHg >= fPresionAnterior - 1 || fPresion_mmHg <= fPresionAnterior + 1)
						{
							nContador = 0;
							fPresionAnterior = fPresion_mmHg;
						}
						else
						{
							nContador++;
							fPresionAnterior = fPresion_mmHg;
						}
	
						if (nContador = 1)
						{
							fDIA = fPresion_mmHg;
							nSecuencia = 3;
						}
					}
					
					nImprimir++;

					if(nImprimir == 10)	
					{
						//Limpiar el GLCD con el color de brocha
						ATGClear();
					
						//Indica donde se inicia a dibujar
						g_Column = 0;
						g_Line	 = 1;
						
						printf("Midiendo...\n");
						/*printf("%d bits\n", (int) fBits);
						printf("%d mV\n", (int) (fVout * 1000));
						printf("%d kPa\n", (int) fPresion_kPa);
						printf("%d mmHg\n", (int) fPresion_mmHg);*/
						nImprimir = 0;
					}
					ESS_Delay(2);
				}
			}
			ESS_Delay(5000);
			nSecuencia = 0;
		}
		
		//Limpiar el GLCD con el color de brocha
		ATGClear();
		
		//Indica donde se inicia a dibujar
		g_Column = 0;
		g_Line	 = 1;
				
		printf(" *** ***	%d PUL/min\n", nPULxmin);
    	printf("********* 	 		   \n");
		printf(" ******* 	  SYS %d\n", (int) fSYS);
 		printf("  *****	 		       \n");
 	 	printf("   ***   	  DIA %d\n", (int) fDIA);
 	 	printf("    *      			   \n");
 	 		
		ESS_Delay(30000/nPULxmin);
		
		//Limpiar el GLCD con el color de brocha
		ATGClear();
		
		//Indica donde se inicia a dibujar
		g_Column = 0;
		g_Line	 = 1;
		
		printf("        	%d PUL/min\n", nPULxmin);
    	printf("	   	 		       \n");
		printf("            SYS %d\n", (int) fSYS);
 		printf("      	 		       \n");
 	 	printf("            DIA %d\n", (int) fDIA);
 	 	printf("           			   \n");
 	 	
		ESS_Delay(30000/nPULxmin);
	}
}