---
template: blog-post
title: Sensor capacitivo de humedad en tierra
slug: /moisturesensor
date: 2021-01-28 20:35
description: STM32F44RE STM32 Nucleo board capacitive moisture sensor sensor
  capacitivo de humedad ADC
---
## Introducción

Siempre me han gustado las plantas, y siempre me ha gustado la automatización, así que no es extraño que cuando vi el proyecto *Garduino* me entraron ganas de hacer algo parecido. Y en eso me hallo, mi objetivo actual es ir poco a poco implementando pequeños módulos para terminar teniendo un sistema de riego automatizado. Quizás no todos los módulos terminen siendo partes del proyecto final, pero seguro que se le encuentra utilidad mas adelante.

En esta entrada trataré un sensor que nos ayudará a decidir si es necesario regar, el sensor de humedad en tierra, por que sería un poco ridículo darse cuenta que tu sistema riega sobre mojado. 

## Lista de dispositivos usados

* **STM32F44RE Nucleo board**.
* Sensor capacitivo de humedad en tierra v2.0

## Enlaces

* [Repositorio Github](https://github.com/Vitalol/SoilMoistureSensorSTM32)

## Consideraciones previas

Se puede encontrar principalmente dos tipos de sensores de humedad para tierra, resistivo y capacitivos. Yo voy a usar el tipo capacitivo, porque el tipo resistivo al tratarse de un conductor expuesto al aire y la humedad termina oxidándose. Ambos son baratos y se pueden encontrar por solo un par de euros.

El sensor capacitivo funciona por que al variar el medio (dieléctrico) en el que se encuentra cambia la capacitancia, el sensor y la circuitería que tiene traduce está variación en una variación de voltaje. Aquí entra en juego el convertidor analógico a digital (ADC) del que dispone nuestro microcontrolador, que se encargará de transformar ese voltaje en un valor digital, por lo que únicamente es necesario configurar un ADC en nuestra placa e interpretar la lectura.

## Configurar ADC

Para configurar el ADC primero es necesario saber que queremos que haga. 

La humedad del suelo no es un aspecto que necesite una medición continua, con tomar una medida cada cierto tiempo es suficiente, por lo que no es necesario activar ninguna forma de escaneo continuo o ADM. 

También necesitamos saber que precisión cumple con nuestras especificaciones, las opciones disponibles son 6b, 8b, 10b o 12b. Si queremos representar la humedad en porcentaje  son necesario por lo menos 7 bits ya que 2<sup>7</sup> = 128 y tendríamos de sobra para representar 100 valores ¿Verdad? Pues resulta que no es del todo cierto y que el sensor no nos proporciona valores entre 0 y 128, ya que tiene un margen por debajo y por encima, por lo que es mejor usar una mayor precisión anqué se pierda al transformarlo en porcentaje.

¿Como vamos a activarlo? existe la opción de activar el ADC de varias maneras, en este ejemplo se va a hacer por software.

Aprovechando la generación de código automática de *STM32CubeIDE* podemos hacer todas estas configuraciones y al final se obtiene un código tal que así:

```c
static void MoisSen_ADC_Init(void)
{

  /* USER CODE BEGIN ADC1_Init 0 */

  /* USER CODE END ADC1_Init 0 */

  ADC_ChannelConfTypeDef sConfig = {0};

  /* USER CODE BEGIN ADC1_Init 1 */

  /* USER CODE END ADC1_Init 1 */
  /** Configure the global features of the ADC (Clock, Resolution, Data Alignment and number of conversion)
  */
  MoisSen.Instance = ADC1;
  MoisSen.Init.ClockPrescaler = ADC_CLOCK_SYNC_PCLK_DIV4;
  MoisSen.Init.Resolution = ADC_RESOLUTION_;
  MoisSen.Init.ScanConvMode = DISABLE;
  MoisSen.Init.ContinuousConvMode = DISABLE;
  MoisSen.Init.DiscontinuousConvMode = DISABLE;
  MoisSen.Init.ExternalTrigConvEdge = ADC_EXTERNALTRIGCONVEDGE_NONE;
  MoisSen.Init.ExternalTrigConv = ADC_SOFTWARE_START;
  MoisSen.Init.DataAlign = ADC_DATAALIGN_RIGHT;
  MoisSen.Init.NbrOfConversion = 1;
  MoisSen.Init.DMAContinuousRequests = DISABLE;
  MoisSen.Init.EOCSelection = ADC_EOC_SINGLE_CONV;
  if (HAL_ADC_Init(&MoisSen) != HAL_OK)
  {
    Error_Handler();
  }
  /** Configure for the selected ADC regular channel its corresponding rank in the sequencer and its sample time.
  */
  sConfig.Channel = ADC_CHANNEL_0;
  sConfig.Rank = 1;
  sConfig.SamplingTime = ADC_SAMPLETIME_3CYCLES;
  if (HAL_ADC_ConfigChannel(&MoisSen, &sConfig) != HAL_OK)
  {
    Error_Handler();
  }
  /* USER CODE BEGIN ADC1_Init 2 */

  /* USER CODE END ADC1_Init 2 */

}
```

Hay que tener en cuenta que este no es el único fragmento de código que configura nuestro ADC, al llamar a la función **HAL_ADC_Init** esta llama a su vez a la función **HAL_ADC_MspInit**.

```c
void HAL_ADC_MspInit(ADC_HandleTypeDef* hadc)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};
  if(hadc->Instance==ADC1)
  {
  /* USER CODE BEGIN ADC1_MspInit 0 */

  /* USER CODE END ADC1_MspInit 0 */
    /* Peripheral clock enable */
    __HAL_RCC_ADC1_CLK_ENABLE();

    __HAL_RCC_GPIOA_CLK_ENABLE();
    /**ADC1 GPIO Configuration
    PA0-WKUP     ------> ADC1_IN0
    */
    GPIO_InitStruct.Pin = GPIO_PIN_0;
    GPIO_InitStruct.Mode = GPIO_MODE_ANALOG;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

  /* USER CODE BEGIN ADC1_MspInit 1 */

  /* USER CODE END ADC1_MspInit 1 */
  }

}
```

Esta función es la que termina configurando los pines y relojes necesarios para que funcione el ADC.

## Obteniendo los datos del ADC

Una vez tenemos configurado el ADC disponemos de tres funciones para iniciarlo, esperar a que esté hecha la conversión y obtener el valor.

```c
	HAL_ADC_Start(&MoisSen);
	HAL_ADC_PollForConversion(&MoisSen, HAL_MAX_DELAY);
	sensor_data = HAL_ADC_GetValue(&MoisSen);
```

Este dato es la información "cruda", un valor entre 0 y 4095 que representa la tensión leída entre 0 y 3,3V. Como es necesario saber los valores para nuestro 0% de humedad en tierra y 100% se puede hacer dos medidas, una al aire (que suponemos que es seco) y otra dentro de un vaso de agua (que suponemos tiene la mayor concentración de agua posible). Por comodidad he usado el UART que incorpora la placa y conecta con el USB para enviar esta información a mi ordenador.

```c
	char txt[] = "La humedad es: ";
	HAL_UART_Transmit(&huart2, (uint8_t*)txt, strlen(txt), HAL_MAX_DELAY);

	sprintf(msg,"%.0f \r\n", sensor_data);
	HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), HAL_MAX_DELAY);
```

![Post2_ADC_Agua_Aire](/assets/post2_adc_agua_aire.png)

## Interpretando los datos del ADC

Como se puede apreciar los valores varían entre 1409 y 3508, mas o menos, dependiendo de cuando se mide y sobre que esté apoyado el sensor varía la lectura. Si nos creemos (y es mucho creer) lo de que el aire es dará la misma medida que la tierra seca y el agua que una tierra totalmente empapada entonces podemos definir una función que nos relacione el porcentaje de humedad con la lectura del ADC.

```c
humidity = (-100*((sensor_data-1440)/(3600-1440))+100);
```

Al aplicar esta transformación podemos ver como varía el porcentaje al entrar y salir del vaso de agua.

![Post2_ADC_Agua_Aire%_2](/assets/post2_adc_agua_aire-_2.png)

Aprovechando el código que escribí para la pantalla LCD (ver [enlace](https://victortamayo.com/LCD16x2_STM32)) puedo ver este porcentaje sin necesitad de tener conectado el microcontrolador a mi pc.

```c
	LCD_clear(&LCD16x2_CfgParam);
	LCD_write_string(&LCD16x2_CfgParam,"Humedad");
	LCD_set_cursor(&LCD16x2_CfgParam, 5, 2);
	sprintf(msg,"%.2f \r\n", humidity);
	HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), HAL_MAX_DELAY);
	LCD_write_string(&LCD16x2_CfgParam,msg);
```

## Discusión sobre la calibración del sensor

Como he comentado antes suponer que el aire y la tierra seca o un vaso de agua y la tierra empapada van a dar el mismo valor es una suposición muy arriesgada, es verdad que podemos esperar que el valor de la tierra esté entre esos dos valores por que no va a estar mas seca que el aire o mas mojada que el agua. Pero si quisiéramos calibrarlo correctamente tendríamos que seguir un procedimiento sencillo pero que consumen mucho tiempo. Tendríamos que partir de una cantidad de tierra seca e ir añadiendo agua poco a poco y esperar a que se absorba este agua, de forma que en cada momento sepamos la cantidad de tierra, la cantidad de agua y la lectura del sensor, y esto habría que hacerlo con cada sensor para asegurarse un funcionamiento optimo.

¿Merece la pena dedicarle el tiempo a esto? Depende de la aplicación.

Si tienes algo que es muy sensible a los cambios de humedad y que te proporciona mucho dinero si merece la pena. En cambio si vas a hacer como yo y lo conectaras a un hueso de aguacate, una tomatera o algo similar por que has visto un video por internet creo que no es tan necesario una calibración correcta.

Y bueno, si lo vas a usar con un cactus mejor apaga y vámonos...