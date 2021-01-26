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

Hace un tiempo compré un set de Arduino uno que incluía entre otras cosas una pantalla LCD de 2 líneas y 16 columnas en la que se pueden mostrar caracteres en una matriz de 5x11 o 5x8 (dependiendo de la configuración) puntos.

En su día la usé junto a una librería para visualizar la cuenta de un reloj, en ese momento no me paré a pensar en cómo funcionaba este dispositivo.  Recientemente volví a encontrar la pantalla LCD, y está vez tenía en mi posesión  una placa de desarrollo STM32 Nucleo, por lo que decidí que era hora de resolver mi duda inicial: *¿Como hago que esto funcione?*

Para programar el microcontrolador he hecho uso de las funciones HAL de STM32, por lo que el código debería ser fácilmente portable, yo no lo he probado.

## Lista dispositivos usados

* **STM32F446RE Nucleo board**, un microcontrolador con una CPU Arm® Cortex®-M4 32-bit
* **LCD 1602 Module**, la pantalla LCD que incluía el kit de Arduino.
* **AzDelivery Logic Analyzer**, un analizador lógico barato para ayudarme en la depuración.

## Consideraciones previas

El módulo LCD cuenta con un pin de habilitación **E**, según el fabricante este pin debe mantenerse a nivel alto mientras se ejecuta la instrucción, estás instrucciones pueden tardar unos milisegundos o unos microsegundos dependiendo de lo que hagan.

Al usar la librería HAL de STM32 disponemos de la función **HAL\_Delay** que crea esperas de tantos ms como le pidas, pero una espera de un ms para algo que necesita 40-60 μs es pasarse, por lo que necesitaremos una función de apoyo, **delay\_us**, para las esperas más cortas.

Estas esperas pueden solventarse leyendo el estado de la bandera *busy* que proporciona el dispositivo, pero para eso necesitas cambiar el modo de funcionamiento de uno de los pins y termina siendo más trabajoso que simplemente esperar un tiempo prudencial.

<i>¡Importante! Tanto HAL_Delay como delay_us van a ser funciones que bloquean el procesador, no es la mejor manera de implementarlo, pero como no voy a usarlo en un sistema con unos requisitos temporales estrictos lo he hecho así.</i>

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

**timer\_delay\_init** Inicializa el contador que se le indique, en mi caso el contador *TIM6* y se le aplica un pre-escalado  igual a la velocidad del reloj dividido un millón, de esta forma (siempre que el reloj sea mayor a 1 Mhz) el contador avanzará en una unidad cada microsegundo.

Ejemplo: Reloj de 8Mhz (que además es como viene configurado por defecto en esta placa), supondría un preescalador de 8, haciendo las cuentas 8x10<sup>6</sup> Hz / 8 = 10<sup>6</sup> Hz.  El contador se actualizara cada microsegundo.

**delay_us** simplemente vigila el valor de contador hasta que llega al valor deseado, no es buena práctica hacerlo de esta manera porque, como he comentado antes, bloquea el procesador, pero he decidido hacerlos así por queno tengo restricciones de tiempo importantes.

# Continuará

