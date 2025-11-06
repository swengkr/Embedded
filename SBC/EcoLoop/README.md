라즈베리파이 SBC(Single Board Computer) 와 PIC MCU를 활용한 저전력·무중단·지속 운용 시스템을 제작하여 운영 중입니다.

### 구성 요소
- 컴퓨팅 모듈 : 라즈베리파이1 모델 B+
- Wakeup : PIC 12F675
- DC 전원 제어 : MOSFET

![](https://github.com/swengkr/Embedded/blob/main/SBC/EcoLoop/project.jpg)
![](https://github.com/swengkr/Embedded/blob/main/SBC/EcoLoop/project2.jpg)
![](https://github.com/swengkr/Embedded/blob/main/SBC/EcoLoop/project3.jpg)

### 회로 결선도
![](https://github.com/swengkr/Embedded/blob/main/SBC/EcoLoop/circuit_diagram.png)

### 라즈베리파이 소스 코드 (Go)
```go
package main

import "C"
import (
	"errors"
	"fmt"
	"io"
	"log"
	"os"
	"os/exec"
	"periph.io/x/conn/v3/gpio"
	"periph.io/x/conn/v3/gpio/gpioreg"
	"periph.io/x/host/v3"
)

func main() {
    var gGPIO17, gGPIO27 gpio.PinIO
	// periph 초기화
	if _, err := host.Init(); err != nil {
		log.Fatal(err)
	}
	if gGPIO17 = gpioreg.ByName("GPIO17"); gGPIO17 == nil {
		log.Fatal("GPIO17 찾을 수 없음")
	}
	if gGPIO27 = gpioreg.ByName("GPIO27"); gGPIO27 == nil {
		log.Fatal("GPIO27 찾을 수 없음")
	}
	// GPIO 모드 설정(17:출력, 27:입력)
	gGPIO17.Out(gpio.Low)
	gGPIO27.In(gpio.PullUp, gpio.NoEdge)
    ...
    for {
        // *** 주기적인 작업 실행 ***
        ...
        // Sleep 시작
        if gGPIO27.Read() == gpio.High {
            // PIC MCU 에 Sleep 요청
            gGPIO17.Out(gpio.High)
            // 리눅스 Shutdown 실행
            cmd := exec.Command("shutdown", "now")
            if _, err := cmd.CombinedOutput(); err != nil {
                log.Fatalf("Shutdown 실패: %v", err)
            } else {
                return
            }
        }
    }
}
```

### PIC MCU 소스 코드 (MPLAB X IDE)
```c
#include <xc.h>

#pragma config FOSC = INTRCIO
#pragma config WDTE = OFF
#pragma config PWRTE = ON
#pragma config MCLRE = OFF
#pragma config BOREN = OFF
#pragma config CP = OFF
#pragma config CPD = OFF

#define _XTAL_FREQ          4000000UL

#define MOSFET              GP0
#define SBC_SHUTDOWN        GP1
#define ALWAYS_ON           GP5

#define CHECK_PERIOD_MS     100UL
#define COUNTS_30_SECONDS   300UL

#define MINUTE_COUNT        600UL
#define HALF_HOUR_COUNT     (MINUTE_COUNT * 30UL)

volatile unsigned long delay_count = 0;
volatile unsigned char is_shutting_down = 0;

void Initialize(void)
{
    CMCON = 0x07;
    ANSEL = 0x00;

    TRISIO0 = 0;
    TRISIO1 = 1;
    TRISIO5 = 1;

    MOSFET = 1;
    INTCON = 0x00;
}

void main(void)
{
    Initialize();

    while (1)
    {
        if (ALWAYS_ON == 0) {
            MOSFET = 1;
        } else if (is_shutting_down == 0) {
            if (SBC_SHUTDOWN == 1) {
                is_shutting_down = 1;
                delay_count = 0;
            }
        } else if (++delay_count >= COUNTS_30_SECONDS) {
            MOSFET = 0;
            __delay_ms(CHECK_PERIOD_MS);
            for (delay_count = 0; delay_count < HALF_HOUR_COUNT && ALWAYS_ON == 1; delay_count++) {
                __delay_ms(CHECK_PERIOD_MS);
            }
            MOSFET = 1;
            is_shutting_down = 0;
        }
        __delay_ms(CHECK_PERIOD_MS);
    }
}
```
