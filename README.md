# -基于 51 单片机的智能交通灯系统-
一、项目简介
赛题关联：本项目为嵌入式芯片与系统设计大赛海思赛题参赛作品，基于51单片机开发，模拟智能交通灯控制应用。
核心功能：实现交通灯信号灯定时循环切换（含绿、黄、红时序控制）、倒计时动态显示，支持左转优先、紧急通行等扩展功能，适配嵌入式系统硬件设计与控制逻辑需求。
二、运行环境
开发板 / 硬件：基于51单片机开发板（可兼容适配海思赛题嵌入式硬件环境 ），包含51单片机核心模块、LED信号灯电路、数码管显示模块、按键输入电路等。
工具链 / 软件：依赖Keil C51编译器（版本适配51单片机开发 ），需提前准备 51 单片机开发环境，可结合海思赛题要求的嵌入式系统调试工具协同使用。
三、编译 & 运行步骤
1. 编译代码
进入代码存储目录
cd /path/to/51_TrafficLight_Code  
打开 Keil uVision 工程文件
在 Keil 软件中，点击 “Rebuild” 按钮执行编译，生成 .hex 文件 
2. 运行程序
使用 51 单片机编程器（如 STC 下载器 ），将编译生成的 .hex 文件烧录到单片机
 步骤示例：
1. 连接编程器与开发板，打开下载软件（如 STC-ISP ）  
2. 选择对应单片机型号、COM 口，加载 .hex 文件  
3. 点击 “下载/编程” ，完成后开发板自动运行程序  
功能验证
正常运行：观察信号灯按绿→黄→红时序循环切换，数码管同步显示倒计时  
功能测试：按下左转、紧急通行按键，验证对应逻辑（如绿灯延长、方向切换等 ）  
四、代码结构
plaintext 
  # 核心代码区
#include <reg51.h>
#define uchar unsigned char 
#define uint unsigned int 

// 交通灯控制引脚定义
sbit NS_G = P2^0; // 南北绿灯 
sbit NS_Y = P2^1; // 南北黄灯 
sbit NS_R = P2^2; // 南北红灯
sbit EW_R = P2^3; // 东西绿灯 
sbit EW_Y = P2^4; // 东西黄灯 
sbit EW_G = P2^5; // 东西红灯

// 数码管控制引脚定义
sbit smg_NS_1 = P1^0; // 南北十位
sbit smg_NS_2 = P1^1; // 南北个位
sbit smg_EW_1 = P1^2; // 东西十位
sbit smg_EW_2 = P1^3; // 东西个位

// 中断按键定义
sbit key_EXT0 = P3^2; // 外部中断0：延长绿灯
sbit key_EXT1 = P3^3; // 外部中断1：切换方向并保持6秒

// 时间设置（简化为统一绿灯、黄灯时间）
uchar GREEN_TIME = 15;  // 绿灯默认时间
uchar YELLOW_TIME = 5;  // 黄灯时间

// 倒计时变量
uchar NS_Time;  // 南北剩余时间
uchar EW_Time;  // 东西剩余时间

uchar ext_flag = 0;     // 中断标志：0-正常 1-延长 2-切换方向
uchar ext_timer = 0;    // 中断计时，应急模式用
uchar saved_state = 0;  // 保存的原状态
uchar saved_NS_Time = 0;// 保存的南北方向时间
uchar saved_EW_Time = 0;// 保存的东西方向时间

// 系统状态（0-南北绿 1-南北黄 2-东西绿 3-东西黄）
uchar state = 0;  
uchar sec_cnt = 0;       // 50ms计数，用于1秒定时

// 共阴极数码管段码表（0~9）
uchar table[] = {0x3F,0x06,0x5B,0x4F,0x66,0x6D,0x7D,0x07,0x7F,0x6F};  

// 毫秒延时（简化版，确保 1ms 精度）
void delay(uint ms) {
    uint i, j;
    for(i = ms; i > 0; i--)
        for(j = 110; j > 0; j--); // 12MHz 下精准延时 1ms
}

// 数码管显示（优化扫描，保证显示稳定）
void display(void) {
    // 南北方向十位
    P0 = table[NS_Time / 10]; 
    smg_NS_1 = 0; 
    delay(1); 
    smg_NS_1 = 1;
    // 南北方向个位
    P0 = table[NS_Time % 10]; 
    smg_NS_2 = 0; 
    delay(1); 
    smg_NS_2 = 1;
    
    // 东西方向十位
    P0 = table[EW_Time / 10]; 
    smg_EW_1 = 0; 
    delay(1); 
    smg_EW_1 = 1;
    // 东西方向个位
    P0 = table[EW_Time % 10]; 
    smg_EW_2 = 0; 
    delay(1); 
    smg_EW_2 = 1;
}

// 定时器+中断初始化
void init(void) {
    TMOD = 0x01;         // 定时器0模式1（50ms定时）
    TH0 = (65536 - 50000) / 256;
    TL0 = (65536 - 50000) % 256;
    EA = 1;  
    ET0 = 1; 
    TR0 = 1; // 开总中断+定时器0
    
    // 外部中断初始化（下降沿触发）
    IT0 = 1; 
    EX0 = 1; 
    IT1 = 1; 
    EX1 = 1; 
}

// 定时器0中断：1秒计时+状态驱动
void timer0() interrupt 1 {
    TH0 = (65536 - 50000) / 256;
    TL0 = (65536 - 50000) % 256;
    
    sec_cnt++;
    if(sec_cnt >= 20) { // 20 * 50ms = 1秒
        sec_cnt = 0;
        
        if(ext_flag == 0) { // 正常模式计时
            if(NS_Time > 0) NS_Time--;
            if(EW_Time > 0) EW_Time--;
        } else if(ext_flag == 2) { // 应急模式计时
            if(ext_timer > 0) {
                ext_timer--;
                NS_Time = ext_timer;
                EW_Time = ext_timer;
            } else {
                // 恢复原状态
                ext_flag = 0;
                state = saved_state;
                NS_Time = saved_NS_Time;
                EW_Time = saved_EW_Time;
            }
        }
    }
}

// 外部中断0（P3.2）：延长绿灯
void ext0() interrupt 0 {
    delay(20); // 消抖
    if(ext_flag == 0 && !key_EXT0) { 
        if((state == 0 || state == 2) && (state == 0 ? NS_Time : EW_Time) < 10) {
            if(state == 0) { // 南北绿灯延长
                NS_Time = 12; 
                EW_Time = 17; // 另一方向红灯时间13s
            } else { // 东西绿灯延长
                EW_Time = 12; 
                NS_Time = 17; 
            }
        }
    }
    while (!key_EXT0); // 等待按键释放，避免重复触发
}

// 外部中断1（P3.3）：应急切换方向
void ext1() interrupt 2 {
    delay(20); // 消抖
    if(ext_flag == 0 && !key_EXT1) { 
        // 保存当前状态和时间
        saved_state = state;
        saved_NS_Time = NS_Time;
        saved_EW_Time = EW_Time;
        
         // 计算应急模式下的目标状态（确保是绿灯状态）
        if(state == 0 || state == 1) { // 原南北方向(绿/黄)→切换到东西绿
            state = 2;
        } else if(state == 2 || state == 3) { // 原东西方向(绿/黄)→切换到南北绿
            state = 0;
        }
        // 应急模式参数设置
        ext_timer = 6; 
        ext_flag = 2; 
        NS_Time = ext_timer;
        EW_Time = ext_timer;
    }
    while(!key_EXT1); // 等待按键释放，避免重复触发
}

// 交通灯状态机（正常模式 + 应急模式）
void traffic_control(void) {
    if(ext_flag == 0) { // 正常循环
        switch(state) {
            case 0: // 南北绿，东西红
                NS_G = 0; NS_Y = 1; NS_R = 1;
                EW_G = 1; EW_Y = 1; EW_R = 0;
                if(NS_Time == 0) { 
                    state = 1; 
                    NS_Time = YELLOW_TIME; 
                }
                break;
            case 1: // 南北黄，东西红
                NS_G = 1; NS_Y = 0; NS_R = 1;
                EW_G = 1; EW_Y = 1; EW_R = 0;
                if(NS_Time == 0) { 
                    state = 2; 
                    EW_Time = GREEN_TIME; 
										NS_Time = GREEN_TIME + YELLOW_TIME;
                }
                break;
            case 2: // 东西绿，南北红
                NS_G = 1; NS_Y = 1; NS_R = 0;
                EW_G = 0; EW_Y = 1; EW_R = 1;
                if(EW_Time == 0) { 
                    state = 3; 
                    EW_Time = YELLOW_TIME; 
                }
                break;
            case 3: // 东西黄，南北红
                NS_G = 1; NS_Y = 1; NS_R = 0;
                EW_G = 1; EW_Y = 0; EW_R = 1;
                if(EW_Time == 0) { 
                    state = 0; 
                    NS_Time = GREEN_TIME; 
										EW_Time = GREEN_TIME + YELLOW_TIME;
                }
                break;
        }
    } else if(ext_flag == 2) { // 应急模式固定状态
        switch(state) {
            case 0: // 应急南北绿
                NS_G = 0; NS_Y = 1; NS_R = 1;
                EW_G = 1; EW_Y = 1; EW_R = 0;
                break;
            case 2: // 应急东西绿
                NS_G = 1; NS_Y = 1; NS_R = 0;
                EW_G = 0; EW_Y = 1; EW_R = 1;
                break;
            default: // 确保状态合法
                state = 0; // 默认南北绿
                NS_G = 0; NS_Y = 1; NS_R = 1;
                EW_G = 1; EW_Y = 1; EW_R = 0;
                break;
        }
    }
}
 
 main.c       # 主程序：交通灯控制逻辑、状态机、定时器中断等
 // 主函数
void main() {
    // 初始化默认状态：南北绿15s，东西红灯（15 + 5 = 20s）
    state = 0; 
    NS_Time = GREEN_TIME; 
    EW_Time = GREEN_TIME + YELLOW_TIME; 
    init(); // 定时器+中断初始化
    
    while(1) {
        display();         // 持续扫描数码管，保证显示
        traffic_control(); // 状态机驱动逻辑
    }
}

//driver.c # 硬件驱动：数码管显示驱动、LED 控制、按键检测  
// 共阴极数码管段码表（0~9）
uchar table[] = {0x3F,0x06,0x5B,0x4F,0x66,0x6D,0x7D,0x07,0x7F,0x6F};  

// 数码管动态扫描显示函数（1ms刷新一次）
void display(void) {
    // 南北方向十位
    P0 = table[NS_Time / 10]; 
    smg_NS_1 = 0;  // 选通南北十位数码管
    delay(1);      // 延时1ms显示
    smg_NS_1 = 1;  // 关闭当前位选通
    
    // 南北方向个位
    P0 = table[NS_Time % 10]; 
    smg_NS_2 = 0;  
    delay(1); 
    smg_NS_2 = 1;
    
    // 东西方向十位
    P0 = table[EW_Time / 10]; 
    smg_EW_1 = 0;  
    delay(1); 
    smg_EW_1 = 1;
    
    // 东西方向个位
    P0 = table[EW_Time % 10]; 
    smg_EW_2 = 0;  
    delay(1); 
    smg_EW_2 = 1;
}
// 交通灯控制引脚定义（P2口）
sbit NS_G = P2^0; // 南北绿灯  
sbit NS_Y = P2^1; // 南北黄灯  
sbit NS_R = P2^2; // 南北红灯  
sbit EW_R = P2^3; // 东西绿灯  
sbit EW_Y = P2^4; // 东西黄灯  
sbit EW_G = P2^5; // 东西红灯  

// 交通灯状态机控制函数（根据状态切换LED）
void traffic_control(void) {
    if(ext_flag == 0) { // 正常模式
        switch(state) {
            case 0: // 南北绿，东西红
                NS_G = 0; NS_Y = 1; NS_R = 1; // 低电平点亮绿灯
                EW_G = 1; EW_Y = 1; EW_R = 0; // 低电平点亮红灯
                // 状态切换逻辑...
                break;
            // 其他状态（南北黄、东西绿、东西黄）的LED控制逻辑类似，通过电平翻转实现亮灭
            // ...
        }
    } else if(ext_flag == 2) { // 应急模式
        // 强制切换LED状态，如应急南北绿或东西绿
        // ...
    }
}
// 中断按键定义（P3口）
sbit key_EXT0 = P3^2; // 外部中断0：左转优先  
sbit key_EXT1 = P3^3; // 外部中断1：紧急切换  

// 毫秒延时函数（用于按键消抖）
void delay(uint ms) {
    uint i, j;
    for(i = ms; i > 0; i--)
        for(j = 110; j > 0; j--); // 12MHz晶振下精准延时1ms
}

// 外部中断0服务函数（左转优先按键检测）
void ext0() interrupt 0 {
    delay(20); // 20ms软件消抖
    if(ext_flag == 0 && !key_EXT0) { // 仅正常模式响应，且按键按下
        // 绿灯时间延长逻辑...
    }
    while (!key_EXT0); // 等待按键释放，避免重复触发
}

// 外部中断1服务函数（紧急切换按键检测）
void ext1() interrupt 2 {
    delay(20); // 20ms软件消抖
    if(ext_flag == 0 && !key_EXT1) { // 仅正常模式响应，且按键按下
        // 保存状态并切换方向逻辑...
    }
    while(!key_EXT1); // 等待按键释放
}

//interrupt.c  # 中断处理：外部中断（左转、紧急按键 ）服务函数  
// 外部中断0：左转优先，延长绿灯时间
void ext0() interrupt 0 {
    delay(20); // 消抖
    if(ext_flag == 0 && !key_EXT0) { 
        if((state == 0 || state == 2) && (state == 0 ? NS_Time : EW_Time) < 10) {
            if(state == 0) { // 南北绿灯延长至12秒
                NS_Time = 12; 
                EW_Time = 17; // 东西红灯同步延长至17秒（12s绿灯+5s黄灯）
            } else { // 东西绿灯延长至12秒
                EW_Time = 12; 
                NS_Time = 17; 
            }
        }
    }
    while (!key_EXT0); // 等待按键释放
}

// 外部中断1：紧急切换通行方向
void ext1() interrupt 2 {
    delay(20); // 消抖
    if(ext_flag == 0 && !key_EXT1) { 
        // 保存当前状态和时间
        saved_state = state;
        saved_NS_Time = NS_Time;
        saved_EW_Time = EW_Time;
        
        // 切换至对向绿灯状态
        if(state == 0 || state == 1) { // 原南北方向→东西绿
            state = 2;
        } else if(state == 2 || state == 3) { // 原东西方向→南北绿
            state = 0;
        }
        
        // 应急模式设置（6秒倒计时）
        ext_timer = 6; 
        ext_flag = 2; 
        NS_Time = ext_timer;
        EW_Time = ext_timer;
    }
    while(!key_EXT1); // 等待按键释放
}

//# 头文件区  
#ifndef CONFIG_H
#define CONFIG_H

// 硬件引脚定义
sbit NS_G = P2^0; // 南北绿灯
sbit NS_Y = P2^1; // 南北黄灯
sbit NS_R = P2^2; // 南北红灯
sbit EW_R = P2^3; // 东西绿灯
sbit EW_Y = P2^4; // 东西黄灯
sbit EW_G = P2^5; // 东西红灯
sbit smg_NS_1 = P1^0; // 南北十位数码管位选
sbit smg_NS_2 = P1^1; // 南北个位数码管位选
sbit smg_EW_1 = P1^2; // 东西十位数码管位选
sbit smg_EW_2 = P1^3; // 东西个位数码管位选
sbit key_EXT0 = P3^2; // 左转优先按键
sbit key_EXT1 = P3^3; // 紧急切换按键

// 时间参数配置
#define GREEN_TIME 15  // 绿灯默认时间（秒）
#define YELLOW_TIME 5   // 黄灯时间（秒）

// 系统状态定义
#define STATE_NS_GREEN 0  // 南北绿灯
#define STATE_NS_YELLOW 1 // 南北黄灯
#define STATE_EW_GREEN 2  // 东西绿灯
#define STATE_EW_YELLOW 3 // 东西黄灯

// 中断标志定义
#define FLAG_NORMAL 0     // 正常模式
#define FLAG_EXTEND 1     // 绿灯延长（未使用，代码中ext_flag=1未实现）
#define FLAG_EMERGENCY 2  // 紧急切换模式

// 函数声明
void delay(uint ms);
void display(void);
void init(void);
void timer0() interrupt 1;
void ext0() interrupt 0;
void ext1() interrupt 2;
void traffic_control(void);

#endif 
//project/   # 工程文件区
编译配置：
目标芯片：STC89C52RC
晶振频率：12MHz
输出路径：./Objects/
生成 HEX 文件选项已勾选

//Objects/     # 编译中间文件、最终 .hex 输出目录  
中间文件：./Objects/TrafficLight.obj, ./Objects/*.lnk
最终 HEX 文件：./Objects/TrafficLight.hex
工程日志：./Objects/TrafficLight.log

└── README.md  # 项目说明文档
五、其他补充
1. 功能演示
实物效果：
可拍摄开发板运行时信号灯切换、数码管显示、按键交互的视频 / 照片，若上传仓库可通过图床 / 仓库内图片路径插入
演示视频：点击跳转至 B 站 / 视频平台演示（需替换实际链接 ）
2. 遇到的问题 & 解决思路
典型难点：数码管动态扫描出现闪烁、显示不同步问题；按键中断存在误触发。
解决方式：优化数码管扫描频率（调整定时器中断周期 ），利用视觉暂留缓解闪烁；为按键增加硬件消抖电路（或软件延时消抖逻辑 ），过滤干扰信号。
3. 开源协议
本项目采用 MIT 开源协议，允许自由使用、修改、分发代码，需保留版权声明。
