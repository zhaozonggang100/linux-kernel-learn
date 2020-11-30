`mrs x25, esr_el1`：把表示同步异常的状态从esr_el1读到通用寄存器x25

`lsr x24, x25, #ESR_ELx_EC_SHIFT`：把x25通用寄存器中的值左移26位，即ESR_EL1的[31:26]位代表EC（异常class）保存到x24

