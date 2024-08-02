# DSP    
2807X[参考手册](/tms28075ReferenceManual.pdf)
## GPIO

## PIE
### PIE中断管理
DSP中有几十个外设中断，通过使用PIE模块管理中断
将几个外设的中断组合称为`group`,以下为芯片定义的3组group，更多请查看参考手册`3.4.5 PIE Channel Mapping`
![PIE Channel Mapping](/dspTutorialPic/PIE_ChannelMapping.png "PIE Channel Mapping")

首先查看中断信号的传播路径[^1^]
[^1^]:TMS320F2807xTechnical Reference Manual 3.4.3 Interrupt Entry Sequence

![Interrupt Propagation Path](/dspTutorialPic/PIE_InterruptPropagationPath.png "Interrupt Propagation Path")
以下为中断信号发生时的事件序列
```
1. The interrupt is latched in PIEIFRx.y.
2. If PIEIERx.y is set, the interrupt propagates.
3. If PIEACK.x is clear, the interrupt propagates and PIEACK.x is set.
4. The interrupt is latched in IFR.x.
5. If IER.x is set, the interrupt propagates.
6. If INTM is clear, the CPU receives the interrupt.
7. Any instructions in the D2 or later stage of the pipeline are run to completion. Instructions in earlier stages are flushed.
8. The CPU saves the context on the stack.
9. IFR.x and IER.x are cleared. INTM is set. EALLOW is cleared.
10. The CPU fetches the ISR vector from the PIE. PIEIFRx.y is cleared.
11. The CPU branches to the ISR.
```
当`group x`中的外设中断信号`y`来临，中断标志锁存器`PIEIFR x.y Latch`会将信号锁存，如果中断使能寄存器`PIEIER x.y`对应的位使能，则中断继续传递下去,这时该group的`PIEACK`会置位，中断信号进入中断标志锁存器`IFR.x latch`进行锁存，如果该`group`的中断已使能，即`IER.x=1`则信号继续传递，如果全局中断使能INTM为0，则cpu发生中断。
对于PIE的配置，以EPWM2中断为例，该中断位于group3，`INT3.2`,在已完成EPWM中断使能的情况下，使能该组的第2位中断
```  
PieCtrlRegs.PIEIER3.bit.INTx2 = 1;
```

配置中断的回调函数`callbackfunction`
```c
    EALLOW;
    PieVectTable.EPWM2_INT = &epwm2_isr; //function for ADCA interrupt 1
    EDIS;
```
写一个函数
```c
interrupt void epwm2_isr(void)
{
    // Clear INT flag for this timer 清除外设中断的标志位
    EPwm2Regs.ETCLR.bit.INT = 1;
    // Acknowledge this interrupt to receive more interrupts from group 3
    //清除PIE group的标志位
    PieCtrlRegs.PIEACK.all = PIEACK_GROUP3; 

}
```

使能group1和全局中断，这个一般放在最后
``` c
    IER |= M_INT3; //Enable group 1 interrupts
    EINT;  // Enable Global interrupt INTM
    ERTM;  // Enable Global realtime interrupt DBGM

```
### 中断优先级
#### Channel Priority
> For every PIE group, the low number channels in the group have the highest priority. For instance in PIE group 1, channel 1.1 has priority over channel 1.3
> 对于每个PIE group，低位通道的中断有着更高的优先级，例如在group 1，1.1通道比1.3通道有着更高的优先级，当这两个中断同时发生时1.1会被优先响应，当1.1通道中断完成1.3通道中断才会开始响应，记得每次响应中断时清除应答PIEACK  
#### Group Priority
> Generally, the lowest channel in the lowest PIE group has the highest priority. An example of this is channels 1.1 
and 2.1. Those two channels have the highest priority in their respective groups. If the interrupts for those two 
enabled channels happened simultaneously and provided there are no other enabled and pending interrupts, 
channel 1.1 is serviced first by the CPU with channel 2.1 left pending.
> 通常来说最低组的最低通道有最高的优先级。比如说通道1.1和通道2.1，这两个通道的中断在它们的group里都有着最高的优先级，当它们同时发生时通道1.1会比通道1.2优先响应
## ADC

## ePWM

## CAN

## DAC




