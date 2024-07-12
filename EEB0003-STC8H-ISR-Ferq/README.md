# STC8H的定时器中断速率估计

今天我尝试了使用STC8H1K08制作DDS信号源，使用定时器中断来实现角度量步进。在测试中我发现高频上不去。

先把完整代码放上来
```
#include <STC8H.h>
#include <INTRINS.h>

// type: STC8H1K08 TSSOP20
// external osc: 32MHz
#define SYSclk 32000000L

#define base_10Hz 25000
#define base_100Hz 2500
#define base_1kHz 250
#define base_10kHz 25

typedef unsigned char uint8_t;
typedef unsigned int uint16_t;
typedef int int16_t;

uint8_t base; // 基数，0=10Hz, 1=100Hz, 2=1kHz, 3=10kHz
uint8_t dec; // 1~9
uint8_t degree; // 当前的角度位置，0~127

xdata uint8_t sine128[128] = {127, 
131, 135, 138, 142, 146, 149, 153, 157, 160, 163, 167, 170, 173, 176, 179, 182, 
184, 187, 189, 191, 193, 195, 197, 198, 200, 201, 202, 203, 203, 204, 204, 204, 
204, 204, 203, 203, 202, 201, 200, 198, 197, 195, 193, 191, 189, 187, 184, 182, 
179, 176, 173, 170, 167, 163, 160, 157, 153, 149, 146, 142, 138, 135, 131, 127, 
123, 119, 116, 112, 108, 105, 101, 97, 94, 91, 87, 84, 81, 78, 75, 72, 
70, 67, 65, 63, 61, 59, 57, 56, 54, 53, 52, 51, 51, 50, 50, 50, 
50, 50, 51, 51, 52, 53, 54, 56, 57, 59, 61, 63, 65, 67, 70, 72, 
75, 78, 81, 84, 87, 91, 94, 97, 101, 105, 108, 112, 116, 119, 123}; // 128点正弦波采样

void Global_Init(void) // 配置IO口并使能外部晶振
{
    P0M0 = 0x00; P0M1 = 0xff; 
    P1M0 = 0x00; P1M1 = 0xff; 
    P2M0 = 0x00; P2M1 = 0xff;
    P3M0 = 0xff; P3M1 = 0x00; // P3推挽输出
    P4M0 = 0x00; P4M1 = 0xff; 
    P5M0 = 0xff; P5M1 = 0x00; 

    P_SW2 = 0x80;
    XOSCCR = 0xc0;                              //启动外部晶振
    while (!(XOSCCR & 1));                      //等待时钟稳定
    CLKDIV = 0x01;                              //时钟不分频
    CLKSEL = 0x01;                              //选择外部晶振
    // MCLKOCR = 0x7f;  // 测试时钟（127分频） 250kHz左右
}

void Timer0_Init(void) // 使能定时器中断并打开总中断
{
    uint16_t div;
    AUXR |= 0x80; // 定时器时钟1T模式
    TMOD &= 0xF0; // 设置定时器模式
    // 设置定时初始值
    if (base == 0)
    {
        div = 65536-base_10Hz;
        TH0 = div / 256;
        TL0 = div % 256;
    }
    else if (base == 1)
    {
        div = 65536-base_100Hz;
        TH0 = div / 256;
        TL0 = div % 256;
    }
    else if (base == 2)
    {
        div = 65536-base_1kHz;
        TH0 = div / 256;
        TL0 = div % 256;
    }    
    else if (base == 3)
    {
        div = 65536-base_10kHz;
        TH0 = div / 256;
        TL0 = div % 256;
    }
    else if (base == 4) // just to test max rate
    {
        div = 65536-50;
        TH0 = div / 256;
        TL0 = div % 256;
    }
    TF0 = 0; // 清除TF0标志
    TR0 = 1; // 定时器0开始计时
    ET0 = 1;
    EA = 1;
}

void Timer0_Interrupt() interrupt 1
{
    degree += dec;
    degree %= 128;
    P3 = sine128[degree];
}

uint8_t Read_config_Pin() // 读取P0端的6个配置脚
{
    /*
    dec = P0 & 0x0f; // 取低端4位
    base = P0 & 0x30;
    base = base >> 4; // 取[4:5]作为基数频率
    */
    
    base = 4;
    dec = 1;
    
    if (dec >= 1 && dec <= 9)
    {
        return 0;
    }
    else
    {
        return 1;
    }
}

void main()
{
    Global_Init();
    while(Read_config_Pin());
    Timer0_Init();
    while (1)
    {
    }
}
```

系统时钟为32MHz，外部晶振。

当定时器中断只有25个时钟的时候，步进速率为1，全周期128点，输出信号的频率为7.5kHz，但理论输出频率：
$$Freq=\frac{32\mathrm{MHz}}{25}/128*1=10\mathrm{kHz}$$

![](pic.jpg)

看起来一次中断无法完成就进入了下一次，放宽定时器中断速率，增加到50个时钟，可以看到输出频率与理论值5kHz相符。

俺寻思中断函数也不需要这么多时钟，这么看来，STC8H的定时器中断速率应该在
$$5\mathrm{kHz}*128=640\mathrm{kHz}$$
到
$$10\mathrm{kHz}*128=1280\mathrm{kHz}$$
之间

~~ps.总算把TSSOP20的STC8H1K08炫光了，早就看TSSOP20不顺眼了，封装这么大引脚经常性连锡，感觉不如QFN~~
