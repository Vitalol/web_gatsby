---
template: blog-post
title: Usar un LCD16x2 con una placa de desarrollo STM32 Nucleo.
slug: /LCD16x2_STM32
date: 2021-01-26 13:05
description: |-
  LCD16x2
  STM32
  NUCLEO-F446RE
  Embebido
  Embedded
  STM32F446RE 
  LCD 1602 Module
---
## Introducción

Hace un tiempo compré un set de *Arduino Uno* que incluía entre otras cosas una pantalla LCD de 2 líneas y 16 columnas en la que se pueden mostrar caracteres en una matriz de 5x11 o 5x8 (dependiendo de la configuración) puntos, a este tipo de pantalla se le suele llamar LCD16x2 para indicar el numero de columnas y filas.

En su día la usé junto a una librería para visualizar la cuenta de un reloj, en ese momento no me paré a pensar en cómo funcionaba este dispositivo.  Recientemente volví a encontrar la pantalla LCD, y está vez tenía en mi posesión  una placa de desarrollo STM32 Nucleo, por lo que decidí que era hora de resolver mi duda inicial: *¿Como hago que esto funcione?*

Para programar el microcontrolador he hecho uso de las funciones HAL de STM32, por lo que el código debería ser fácilmente portable, yo no lo he probado.

## Lista dispositivos usados

- **STM32F446RE Nucleo board**, un microcontrolador con una CPU Arm® Cortex®-M4 32-bit
- **LCD 1602 Module**, la pantalla LCD que incluía el kit de Arduino.
- **AzDelivery Logic Analyzer**, un analizador lógico barato para ayudarme en la depuración.

## Enlaces

* [Repositorio Github](https://github.com/Vitalol/LCD16x2_DRIVERS)
* [Datasheet](https://datasheetspdf.com/pdf-file/519148/CA/LCD-1602A/1)

## Consideraciones previas

El módulo LCD cuenta con un pin de habilitación **E**, según el fabricante este pin debe mantenerse a nivel alto mientras se ejecuta la instrucción, estás instrucciones pueden tardar unos milisegundos o unos microsegundos dependiendo de lo que hagan.

Al usar la librería HAL de STM32 disponemos de la función **HAL_Delay** que crea esperas de tantos ms como le pidas, pero una espera de un ms para algo que necesita 40-60 μs es pasarse, por lo que necesitaremos una función de apoyo, **delay_us**, para las esperas más cortas.

Estas esperas pueden solventarse leyendo el estado de la bandera *busy* que proporciona el dispositivo, pero para eso necesitas cambiar el modo de funcionamiento de uno de los pins y termina siendo más trabajoso que simplemente esperar un tiempo prudencial.

*¡Importante! Tanto HAL_Delay como delay_us van a ser funciones que bloquean el procesador, no es la mejor manera de implementarlo, pero como no voy a usarlo en un sistema con unos requisitos temporales estrictos lo he hecho así.*

El controlador del LCD (*HD44780* o similar) dispone de dos modos de funcionamiento, bus de dato de 8 bits o de 4 bits, en el segundo caso tienes que enviar a información en dos veces, pero a cambio tienes que usar 4 cables menos. Este es el modo de funcionamiento que se ha implementado.

## Delay_us

Esta función se puede encontrar en la carpeta utilities dentro el proyecto.

En la cabecera se encuentran los prototipos de las funciones.

```c
#ifndef DELAY_H_
#define DELAY_H_
#include "stm32f4xx_hal.h"
#include "main.h"

void timer_delay_init(void);
void delay_us(volatile uint16_t u16);


#endif /* DELAY_H_ */
```

En el código fuente se encuentran la implementación de las dos funciones.

```c
#include "delay.h"
TIM_HandleTypeDef HTIMx;
uint32_t gu32_ticks = 0;

void timer_delay_init(void){

    uint32_t gu32_ticks = (HAL_RCC_GetHCLKFreq() / 1000000);

    HTIMx.Instance = TIM6;
    HTIMx.Init.CounterMode = TIM_COUNTERMODE_UP;
    HTIMx.Init.ClockDivision = TIM_CLOCKDIVISION_DIV1;
    HTIMx.Init.Prescaler = gu32_ticks-1;
    HTIMx.Init.Period = 0xFFFF;

    __HAL_RCC_TIM6_CLK_ENABLE();

    if (HAL_TIM_Base_Init(&HTIMx)!=HAL_OK){
        Error_Handler();
}
    HAL_TIM_Base_Start(&HTIMx);
}

void delay_us(volatile uint16_t u16){

    HTIMx.Instance->CNT = 0;
    while(HTIMx.Instance->CNT<=u16){
    }

}
```

**timer_delay_init** Inicializa el contador que se le indique, en mi caso el contador *TIM6* y se le aplica un pre-escalado  igual a la velocidad del reloj dividido un millón, de esta forma (siempre que el reloj sea mayor a 1 Mhz) el contador avanzará en una unidad cada microsegundo.

Ejemplo: Reloj de 8Mhz (que además es como viene configurado por defecto en esta placa), supondría un pre-escalado de 8, haciendo las cuentas 8x106 Hz / 8 = 106 Hz.  El contador se actualizara cada microsegundo.

**delay_us** simplemente vigila el valor de contador hasta que llega al valor deseado, no es buena práctica hacerlo de esta manera porque, como he comentado antes, bloquea el procesador, pero he decidido hacerlos así por que no tengo restricciones de tiempo importantes.

# LCD16x2

Para controlar la pantalla LCD son necesarias entre otras cosas: configurar los pines de entrada y salida (GPIO),  mandarle una secuencia de inicio mediante estos pines y mandarle *instrucciones* para que haga cosas como mostrar un carácter, mover el cursor o limpiar la pantalla. A continuación se describe como he implementado cada parte. Recuerdo que todo está pensado para la modalidad de bus de datos de 4 bits.

### Configuración GPIO

Me parece importante que el usuario sea quien decide que pines usar para conectarse a la pantalla LCD, por lo que he empezado definiendo una estructura donde guardar la información necesaria.

```c
typedef struct
{
	GPIO_TypeDef *LCD_GPIO;
	uint16_t D4_PIN;
	uint16_t D5_PIN;
	uint16_t D6_PIN;
	uint16_t D7_PIN;
	uint16_t EN_PIN;
	uint16_t RW_PIN;
	uint16_t RS_PIN;
	uint16_t LCD_EN_Delay;

}LCD16x2_CfgType;
```

Usando esta estructura y  la estructura **GPIO_InitTypdef** de las librerías HAL se puede implementar una función que configura los GPIO de acuerdo a lo definido en la estructura **LCD16x2_CfgType**.

```c
void LCD_GPIO_cfg(LCD16x2_CfgType *LCD16x2_CfgParam){

	if(LCD16x2_CfgParam->LCD_GPIO == GPIOA)
		__HAL_RCC_GPIOA_CLK_ENABLE();
	if(LCD16x2_CfgParam->LCD_GPIO == GPIOB)
		__HAL_RCC_GPIOB_CLK_ENABLE();
	if(LCD16x2_CfgParam->LCD_GPIO == GPIOC)
		__HAL_RCC_GPIOC_CLK_ENABLE();
	if(LCD16x2_CfgParam->LCD_GPIO == GPIOD)
		__HAL_RCC_GPIOD_CLK_ENABLE();

	GPIO_InitTypeDef LCD_GPIO;

	LCD_GPIO.Pin = LCD16x2_CfgParam->D7_PIN | LCD16x2_CfgParam->D6_PIN | LCD16x2_CfgParam->D5_PIN | LCD16x2_CfgParam->D4_PIN | LCD16x2_CfgParam->EN_PIN | LCD16x2_CfgParam->RS_PIN | LCD16x2_CfgParam->RW_PIN;
	LCD_GPIO.Mode = GPIO_MODE_OUTPUT_PP;
	LCD_GPIO.Pull = GPIO_NOPULL;
	LCD_GPIO.Speed = GPIO_SPEED_LOW;
	HAL_GPIO_Init(LCD16x2_CfgParam->LCD_GPIO, &LCD_GPIO);

}
```

Esta función hay que llamarla después de asignar los pines en la estructura tipo**LCD16x2_CfgType**, a continuación se muestra como lo he hecho yo.

```c
void LCD16x2_Config(void){

	// LCD Cfg struct
    
	LCD16x2_CfgParam.LCD_GPIO = GPIOB;

	LCD16x2_CfgParam.D7_PIN = GPIO_PIN_13;
	LCD16x2_CfgParam.D6_PIN = GPIO_PIN_5;
	LCD16x2_CfgParam.D5_PIN = GPIO_PIN_4;
	LCD16x2_CfgParam.D4_PIN = GPIO_PIN_10;

	LCD16x2_CfgParam.EN_PIN = GPIO_PIN_14;
	LCD16x2_CfgParam.RS_PIN = GPIO_PIN_1;
	LCD16x2_CfgParam.RW_PIN = GPIO_PIN_15;

	LCD16x2_CfgParam.LCD_EN_Delay = 60;

	LCD_GPIO_cfg(&LCD16x2_CfgParam, &LCD_GPIO);
    
}
```

**LCD16x2_CfgParam** está definido a nivel global en *main.c*,  después creo una función **LCD16x2_Config** que debe llamarse antes de la función encargada de la secuencia de iniciación y se encarga de configurar los pines indicados como pines de salida.

### Secuencia de iniciación

Esta es posiblemente la parte mas sensible, puesto que hay que seguir un orden muy especifico que viene indicado en el datasheet, durante esta secuencia se deciden cosas como el el tamaño de los caracteres, el parpadeo del cursor o la dirección del movimiento del cursor.

Para realizar esta secuencia me he apoyado en la función **LCD_cmd**, que dado una instrucción modifica el estado de los pines según sea necesario y manda un pulso en el pin de *enable*. Estas instrucciones son de 6 bits por que recordemos que tenemos RS, RW y de D7 a D4 en nuestro bus.

```c
void LCD_cmd(LCD16x2_CfgType *LCD16x2_CfgParam, uint8_t Inst){
	LCD_pin_set(LCD16x2_CfgParam, Inst);
	HAL_GPIO_WritePin(LCD16x2_CfgParam->LCD_GPIO, LCD16x2_CfgParam->EN_PIN, 1);
	delay_us(LCD16x2_CfgParam->LCD_EN_Delay);
	HAL_GPIO_WritePin(LCD16x2_CfgParam->LCD_GPIO, LCD16x2_CfgParam->EN_PIN,0);
	delay_us(LCD16x2_CfgParam->LCD_EN_Delay);
}
```

Esta a su vez utiliza la función **LCD_pin_set** para poner las señales necesarias en los pines de salida.

```c
static void LCD_pin_set(LCD16x2_CfgType *LCD16x2_CfgParam, int8_t Instr){
	HAL_GPIO_WritePin(LCD16x2_CfgParam->LCD_GPIO,LCD16x2_CfgParam->RS_PIN, ((Instr)&(1<<5))>>5);
	HAL_GPIO_WritePin(LCD16x2_CfgParam->LCD_GPIO,LCD16x2_CfgParam->RW_PIN, ((Instr)&(1<<4))>>4);
	HAL_GPIO_WritePin(LCD16x2_CfgParam->LCD_GPIO,LCD16x2_CfgParam->D7_PIN, ((Instr)&(1<<3))>>3);
	HAL_GPIO_WritePin(LCD16x2_CfgParam->LCD_GPIO,LCD16x2_CfgParam->D6_PIN, ((Instr)&(1<<2))>>2);
	HAL_GPIO_WritePin(LCD16x2_CfgParam->LCD_GPIO,LCD16x2_CfgParam->D5_PIN, ((Instr)&(1<<1))>>1);
	HAL_GPIO_WritePin(LCD16x2_CfgParam->LCD_GPIO,LCD16x2_CfgParam->D4_PIN, (Instr)&(1));
}
```

El funcionamiento es bastante sencillo, enmascaro cada pin con su posición en el código de instrucción y desplazo el bit resultante a la posición 0, de forma que se obtiene el valor del bit de esa posición y puedo usarlo para poner la salida a nivel bajo o alto.

Con estas dos funciones definidas puedo por fin comunicarme con la pantalla LCD, a la que le mando las instrucciones según me interesa que se inicialice.

```c
void LCD_init(LCD16x2_CfgType *LCD16x2_CfgParam){

	//Power on
	// WAIT 15 ms to be sure the LCD has Power on correctly
	HAL_Delay(150);
	// Ini instruction as given in datasheet
	LCD_cmd(LCD16x2_CfgParam, LCD_fun_set_ini);
	HAL_Delay(5);
	LCD_cmd(LCD16x2_CfgParam, LCD_fun_set_ini);
	delay_us(150);
	LCD_cmd(LCD16x2_CfgParam, LCD_fun_set_ini);
	LCD_cmd(LCD16x2_CfgParam, LCD_fun_set_4bits);
	LCD_cmd(LCD16x2_CfgParam, LCD_fun_set_lines_font_1);
	LCD_cmd(LCD16x2_CfgParam, LCD_fun_set_lines_font_2);
	LCD_cmd(LCD16x2_CfgParam, LCD_dp_off_1);
	LCD_cmd(LCD16x2_CfgParam, LCD_dp_off_2);
	LCD_cmd(LCD16x2_CfgParam, LCD_dp_clr_1);
	LCD_cmd(LCD16x2_CfgParam, LCD_dp_clr_2);
	LCD_cmd(LCD16x2_CfgParam, LCD_mode_set_1);
	LCD_cmd(LCD16x2_CfgParam, LCD_mode_set_2);

}
```

En el caso del ejemplo el cursor no aparece, se activan las dos filas y cada carácter ocupa 8 puntos de alto. Estas instrucciones pueden editarse en el cabecero asociado. Además las instrucciones han sido divididas en dos instrucciones, por que estamos implementando el modo de 4  bits.

### Escribir un carácter

Para escribir un carácter en pantalla basta con "acceder" a la dirección de memoria asociado a dicho carácter, además da la casualidad de que están ordenados en orden ASCII, por lo que no hay que hacer ninguna adaptación. Nuestro único problema es que nuestro bus de datos es de 4 bits y cada carácter ocupa 8 bits, por lo que tendremos que dividirlos en dos conjuntos de 4 bits (o Nibble).

```c
void LCD_write_char(LCD16x2_CfgType *LCD16x2_CfgParam, char Data){

	// Divide the 8 bit caracters in two 2 bits
	uint8_t LowNibble, HighNibble;

	LowNibble = Data&0x0F;
	HighNibble = Data&0xF0;
	HighNibble = (HighNibble>>4);

	uint8_t address = 0b100000; //Used to complete instruction to 6bits

	LowNibble |= address;
	HighNibble |= address;
	
	LCD_cmd(LCD16x2_CfgParam, HighNibble);
	LCD_cmd(LCD16x2_CfgParam, LowNibble);

}
```

El funcionamiento de la función  **LCD_write_char** es sencillo, primero enmascara con la parte que queremos quedarnos, a continuación se asegura de que está en los 4 primeros bits y le añade los bits correspondientes a RW y RS y a continuación manda ambos usando la función **LCD_cmd**.

Para escribir una cadena basta con escribir varios caracteres, el mismo controlador del LCD se encarga de que vaya avanzando el cursor.

```C
void LCD_write_string(LCD16x2_CfgType *LCD16x2_CfgParam, char *string){

	for (int i = 0; string[i]!= '\0'; i++){
		LCD_write_char(LCD16x2_CfgParam, string[i]);
	}
}
```

### Resto de funcionalidades

El resto de funcionalidades pueden verse en el repositorio, no tiene sentido seguir alargando este post, en cada caso solo es necesario usar la función **LCD_cmd** con la instrucción adecuada siguiendo las tablas que ofrece el fabricante.
