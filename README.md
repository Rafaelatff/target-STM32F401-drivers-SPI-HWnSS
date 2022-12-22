# target-STM32F401-drivers-SPI-HWnSS
This project uses STM32CubeIDE and it's a program created to practice my C habilities during the course 'Mastering Microcontroller and Embedded Driver Development' from FastBit Embedded Brain Academy. I am using a NUCLEO-F401RE board.

## nSS controlled by hardware (not sw as code before)

For this exercise we are going to:
B1 says when to send PSI data

SPI Full-duplex mode, (ST - master) === SPI ===  (arduino - slave), DFF = 0, SSM = 0 (NSS controlled by HW), SCLK = 2 Mhz.

 *	DOWNLOAD ARDUINO CODE: https://github.com/niekiran/MasteringMCU

PB14 --> MISO - Not required (Only TX for now)

PB15 --> MOSI

PB13 --> SCLK

PB12 --> NSS

## added to previous code

Implemention of B1 buttom. Note that the code to enable peripheral clock is inside function and there is no need of calling this function again. 

![image](https://user-images.githubusercontent.com/58916022/209189427-be6b3153-08ff-43cd-9433-153cf57408c5.png)

Now, the 16MHz clock of peripheral is divided by 8, resulting in a SPI communication of 2 Mhz. SSM (of CR1 register) is disabled, making that the chip select must be controlled by hardware (not fw anymore).

![image](https://user-images.githubusercontent.com/58916022/209189579-55d615d4-6771-49ae-9ba9-09cffbf28e7a.png)

So, the NSS pin isn't a comment anymore. PB12 is configured as PSI peripheral NSS by the alternate function 5.

![image](https://user-images.githubusercontent.com/58916022/209189969-888c9c10-6c06-4be6-97b1-54bddf54ead9.png)

SSI configuration (to remove error) was deleted (is only required when using FW NSS). But now, we needed to set the SSOE bit from CR2 register. This means, with this configuration, aways that the SPE (Serial Peripheral Enable) is active, the NSS bit will go low (and activate chip select).

![image](https://user-images.githubusercontent.com/58916022/209190282-75ab421c-4141-4073-b150-06e1e574fb25.png)

![image](https://user-images.githubusercontent.com/58916022/209190623-92a5e5f6-2f52-4f1e-bafe-f88f1af3b5d9.png)

In the results, the green line will show the output for the NSS pin according with the line: SPI_PeripheralControl(SPI2,EnOrDi);

After sending data, we set a busy flag to hold the code while all the caracters are being sent by the SPI peripheral. Only after that the SPI can be disabled.

![image](https://user-images.githubusercontent.com/58916022/209190945-c4492d87-b845-45cd-a862-faefaf87dce1.png)

## final code

```
/*
 *
 *  Created on: 22 de dez de 2022
 *      Author: r.freitas
 *
 *      For this exercise we are going to:
 *      B1 says when to send PSI data
 *
 *      SPI Full-duplex mode
 *		(ST - master) === SPI ===  (arduino - slave)
 *			DFF = 0
 *			SSM = 0 (NSS controlled by HW)
 * 			SCLK = 2 Mhz
 *
 *	DOWNLOAD ARDUINO CODE: https://github.com/niekiran/MasteringMCU
 *
 *      PB14 --> MISO - Not required (Only TX for now)
 *      PB15 --> MOSI
 *      PB13 --> SCLK
 *      PB12 --> NSS
 *
 */
#include <string.h> // to use strlen() function
#include "stm32f401xx.h"

void delay(void){
	for (uint32_t i=0; i<500000; i++);
}

void SPI_GPIOInits(void);
void SPI2_Inits(void);
void GPIO_ButtonInit(void);


int main (void){

	char user_data[] = "Aello World"; // cannot pass 256 bytes (uint8_t dataLen)

	SPI_GPIOInits(); //Initialize GPIO pins to behave as SPI2 pins

	GPIO_ButtonInit(); ////Initialize GPIO pin for B1

	SPI2_Inits(); //Initialize the SPI2 peripheral parameters

	// Turn on SSOE to control NSS pin according to SPEnable.
	SPI_SSOEConfig(SPI2,ENABLE);

	while(1){
		while(GPIO_Read(GPIOC, GPIO_PIN_NO_13));
		delay();

		// Enable the SPI2 peripheral
		SPI_PeripheralControl(SPI2,ENABLE);

		// Arduino code '001SPISlaveRxString.ino' expects 1 byte with
		// Size frame information (8 bits or 16 bits), then data
		uint8_t dataLen = strlen(user_data);
		SPI_SendData(SPI2,&dataLen,1);

		SPI_SendData(SPI2,(uint8_t*)user_data,strlen(user_data));

		// Need to wait all bits are sent (SPI Comm not busy) bf disable SPI
		// We use budy flag for that (BSY)
		 while(SPI_GetFlagStatus(SPI2,SPI_BUSY_FLAG)); //pooling, not ideal technique

		// Disable the SPI2 peripheral
		SPI_PeripheralControl(SPI2,DISABLE);
	}
}

void SPI_GPIOInits(void){
	GPIO_Handle_t SPIPins;

	SPIPins.pGPIOx = GPIOB;
	SPIPins.GPIO_PinConfig.GPIO_PinMode = GPIO_MODE_ALTFN;
	SPIPins.GPIO_PinConfig.GPIO_PinAltFunMode = 5;
	SPIPins.GPIO_PinConfig.GPIO_PinOpType = GPIO_OP_TYPE_PP;
	//SPIPins.GPIO_PinConfig.GPIO_PinPuPdControl = GPIO_NO_PUPD;
	SPIPins.GPIO_PinConfig.GPIO_PinPuPdControl = GPIO_PIN_PU;
	SPIPins.GPIO_PinConfig.GPIO_PinSpeed = GPIO_SPEED_FAST;

	//PB13 --> SCLK
	SPIPins.GPIO_PinConfig.GPIO_PinNumber = GPIO_PIN_NO_13;
	GPIO_Init(&SPIPins);

	//PB15 --> MOSI
	SPIPins.GPIO_PinConfig.GPIO_PinNumber = GPIO_PIN_NO_15;
	GPIO_Init(&SPIPins);

	//PB14 --> MISO --> not used, can be commented
	//SPIPins.GPIO_PinConfig.GPIO_PinNumber = GPIO_PIN_NO_14;
	//GPIO_Init(&SPIPins);

	//PB12 --> NSS
	SPIPins.GPIO_PinConfig.GPIO_PinNumber = GPIO_PIN_NO_12;
	GPIO_Init(&SPIPins);

}

void SPI2_Inits(void){
	SPI_Handle_t SPI2Handle;

	SPI2Handle.pSPIx = SPI2;
	SPI2Handle.SPIConfig.SPI_BusConfig = SPI_BUS_CONFIG_FD;
	SPI2Handle.SPIConfig.SPI_DeviceMode = SPI_DEVICE_MODE_MASTER;
	SPI2Handle.SPIConfig.SPI_SclkSpeed = SPI_SCLK_SPEED_DIV8; // Generates SCLK 2MHz
	SPI2Handle.SPIConfig.SPI_DFF = SPI_DFF_8BITS;
	SPI2Handle.SPIConfig.SPI_CPOL = SPI_CPOL_LOW;
	SPI2Handle.SPIConfig.SPI_CPHA = SPI_CPHA_LOW;
	SPI2Handle.SPIConfig.SPI_SSM = SPI_SSM_DI; // HW slave management en for NSS pin

	// With HW NS (SSM =0), when SPE is enable, NSS pin is automatically pulled to low.
	// PB12 --> NSS
	// But for that, the NSS O/P -: Slave Select Output Enable must be set!

	SPI_Init(&SPI2Handle);
}

void GPIO_ButtonInit(void){
	GPIO_Handle_t B1;

	//B1 configuration
	B1.pGPIOx = GPIOC;
	B1.GPIO_PinConfig.GPIO_PinNumber = GPIO_PIN_NO_13;
	B1.GPIO_PinConfig.GPIO_PinMode = GPIO_MODE_IN;
	B1.GPIO_PinConfig.GPIO_PinSpeed = GPIO_SPEED_FAST;
	B1.GPIO_PinConfig.GPIO_PinPuPdControl = GPIO_NO_PUPD; //It has an external PU

	GPIO_Init(&B1);
}

```

## results

"Aello World" string being sent. In green, line corresponding to NSS.

![WhatsApp Image 2022-12-22 at 14 06 55](https://user-images.githubusercontent.com/58916022/209191124-028f27c5-976c-4b93-9118-7b7315acedd9.jpeg)

Checking clock frequency of SPI:

![WhatsApp Image 2022-12-22 at 14 06 40](https://user-images.githubusercontent.com/58916022/209191278-be669d40-23a0-4c0b-bbc8-983c70a56d09.jpeg)




