# TC389 FreeRTOS 任务 CSA 分配机制说明

## 1. CSA 概述

CSA（Context Save Area，上下文保存区）是 TriCore 架构特有的硬件机制，用于在函数调用和中断处理时自动保存和恢复 CPU 上下文信息。

### 1.1 CSA 基本特性

- **每个 CSA 大小**：16 个字（64 字节）
- **CSA 用途**：保存函数调用时的寄存器上下文（PCXI、PSW、A10-A15、D8-D15 等）
- **管理方式**：通过链表结构管理，FCX 寄存器指向空闲 CSA 链表的头部

## 2. TC389 各核心的 CSA 配置

根据链接脚本 `Lcf_Tasking_Tricore_Tc.lsl` 的配置：

| 核心 | CSA 起始地址 | CSA 结束地址 | CSA 大小 | 可容纳 CSA 数量 |
|------|-------------|-------------|----------|----------------|
| CPU0 | 0x70039c00  | 0x7003bc00  | 8 KB     | 128 个         |
| CPU1 | 0x60039c00  | 0x6003bc00  | 8 KB     | 128 个         |
| CPU2 | 0x50015c00  | 0x50017c00  | 8 KB     | 128 个         |
| CPU3 | 0x40015c00  | 0x40017c00  | 8 KB     | 128 个         |

### 2.1 CSA 内存布局

每个核心的 DSRAM 布局（从高地址到低地址）：
```
高地址
  ┌─────────────────┐
  │   CSA (8KB)     │  ← 每个核心独立的 CSA 池
  ├─────────────────┤
  │  ISTACK (1KB)   │  ← 中断栈
  ├─────────────────┤
  │  USTACK (2KB)   │  ← 用户栈
  ├─────────────────┤
  │   HEAP (4KB)    │  ← 堆内存
  └─────────────────┘
低地址
```

## 3. CSA 初始化过程

### 3.1 系统启动时的 CSA 初始化

在系统启动代码中（`Ifx_Ssw_Tc0.c`），会调用 `Ifx_Ssw_initCSA()` 函数初始化 CSA 链表：

```c
Ifx_Ssw_initCSA((unsigned int *)__CSA(0), (unsigned int *)__CSA_END(0));
```

**初始化过程**：
1. 计算 CSA 数量：`numOfCsa = (csaEnd - csaBegin) / 64`
2. 将每个 CSA 链接成链表，每个 CSA 的第一个字（PCXI）指向下一个 CSA
3. 将 FCX 寄存器设置为第一个 CSA 的地址
4. 将 LCX 寄存器设置为倒数第三个 CSA 的地址（用于检测 CSA 耗尽）

## 4. FreeRTOS 任务创建时的 CSA 分配

### 4.1 任务创建流程

当调用 `xTaskCreate()` 创建任务时，会调用 `pxPortInitialiseStack()` 函数初始化任务栈，该函数会分配 CSA。

**关键代码**（`port.c:116-176`）：

```c
StackType_t *pxPortInitialiseStack( StackType_t * pxTopOfStack,
                                    TaskFunction_t pxCode,
                                    void * pvParameters )
{
    uint32_t xLowerCsa = 0, xUpperCsa = 0;
    uint32_t * pxUpperCSA = NULL;
    uint32_t * pxLowerCSA = NULL;

    /* 禁用中断，因为要操作 CSA */
    __disable();
    {
        __dsync();
        
        /* 从空闲 CSA 链表中获取两个 CSA */
        xLowerCsa = __mfcr( portCPU_FCX );        // 获取空闲 CSA 链表头
        pxLowerCSA = pxPortCsaToAddress( xLowerCsa );
        
        if( pxLowerCSA != NULL )
        {
            /* Lower CSA 链接到 Upper CSA */
            xUpperCsa = pxLowerCSA[ 0 ];          // Lower CSA 的 PCXI 指向 Upper CSA
            pxUpperCSA = pxPortCsaToAddress( pxLowerCSA[ 0 ] );
        }
        
        /* 检查是否成功获取两个 CSA */
        if( ( pxLowerCSA != NULL ) && ( pxUpperCSA != NULL ) )
        {
            /* 从空闲链表中移除这两个 CSA */
            __mtcr( portCPU_FCX, pxUpperCSA[ 0 ] );  // 更新 FCX 指向下一个空闲 CSA
        }
        else
        {
            /* CSA 耗尽，触发陷阱 */
            __asm( "\tsvlcx" );
        }
    }
    __enable();
    
    /* 初始化 Upper Context */
    memset( pxUpperCSA, 0, portNUM_WORDS_IN_CSA * sizeof( uint32_t ) );
    pxUpperCSA[ 2 ] = ( uint32_t ) pxTopOfStack;  // A10: 栈指针
    pxUpperCSA[ 1 ] = portINITIAL_SYSTEM_PSW;     // PSW: 程序状态字
    pxUpperCSA[ 0 ] = portINITIAL_UPPER_PCXI;     // PCXI: 上下文链接
    
    /* 初始化 Lower Context */
    memset( pxLowerCSA, 0, portNUM_WORDS_IN_CSA * sizeof( uint32_t ) );
    pxLowerCSA[ 8 ] = ( uint32_t ) pvParameters;  // A4: 参数寄存器
    pxLowerCSA[ 1 ] = ( uint32_t ) pxCode;        // A11: 返回地址（任务入口）
    pxLowerCSA[ 0 ] = portINITIAL_LOWER_PCXI | xUpperCsa;  // PCXI 指向 Upper context
    
    /* 在栈顶保存 Lower CSA 的地址 */
    pxTopOfStack--;
    *pxTopOfStack = 0;  // uxCriticalNesting
    pxTopOfStack--;
    *pxTopOfStack = xLowerCsa;  // Lower CSA 地址
    
    return pxTopOfStack;
}
```

### 4.2 每个任务分配的 CSA 数量

**重要说明**：任务并不拥有"固定分配"的 CSA，而是从全局 CSA 池中动态分配和释放。

**任务创建时**：
- 从全局 CSA 池中分配 **2 个 CSA** 用于初始化任务上下文
  - **Upper CSA**：保存上层上下文（PSW、A10-A15、D8-D15）
  - **Lower CSA**：保存下层上下文（A4-A7、A11、D0-D7）

**任务运行时**：
- 如果任务中有函数调用，TriCore 硬件会自动从全局 CSA 池分配更多 CSA
- 函数返回时，CSA 会自动释放回全局池
- 任务使用的 CSA 数量取决于函数调用深度

**任务切换时**：
- 当前任务使用的 CSA 链状态被保存到任务栈中
- 新任务恢复时，从任务栈恢复 CSA 链状态

**任务删除时**：
- 任务使用的所有 CSA 被回收到全局 CSA 池

### 4.3 CSA 在任务栈中的存储

任务栈顶部的布局：
```
高地址
  ┌─────────────────┐
  │  ...            │
  ├─────────────────┤
  │  xLowerCsa      │  ← Lower CSA 的地址（PCXI 格式）
  ├─────────────────┤
  │  0              │  ← uxCriticalNesting
  ├─────────────────┤
  │  ...            │  ← 任务栈的其他内容
  └─────────────────┘
低地址
```

## 5. 任务切换时的 CSA 使用

### 5.1 保存上下文（`vPortSaveContext`）

当任务切换时，当前任务的 CSA 链会被保存到任务栈中：

```c
void vPortSaveContext( unsigned char ucCallDepth )
{
    /* 获取当前 PCXI（Lower CSA 地址） */
    uxLowerCSA = __mfcr( portCPU_PCXI );
    
    /* 通过 PCXI 链找到 Upper CSA */
    pxLowerCSA = pxPortCsaToAddress( uxLowerCSA );
    pxUpperCSA = pxPortCsaToAddress( pxLowerCSA[ 0 ] );
    
    /* 保存栈指针到 TCB */
    *ppxTopOfStack = ( uint32_t * ) pxUpperCSA[ 2 ];
    
    /* 保存 uxCriticalNesting 和 Lower CSA 地址到栈 */
    ( *ppxTopOfStack )--;
    **ppxTopOfStack = uxCriticalNesting;
    ( *ppxTopOfStack )--;
    **ppxTopOfStack = uxLowerCSA;
}
```

### 5.2 加载上下文（`vPortLoadContext`）

切换到新任务时，从任务栈恢复 CSA：

```c
void vPortLoadContext( unsigned char ucCallDepth )
{
    /* 从栈中恢复 Lower CSA 地址 */
    uxLowerCSA = **ppxTopOfStack;
    ( *ppxTopOfStack )++;
    uxCriticalNesting = **ppxTopOfStack;
    ( *ppxTopOfStack )++;
    
    /* 设置 PCXI 寄存器 */
    __mtcr( portCPU_PCXI, uxLowerCSA );
}
```

## 6. 任务删除时的 CSA 回收

当任务被删除时，需要将任务占用的 CSA 回收到空闲链表：

```c
void vPortReclaimCSA( unsigned long ** pxTCB )
{
    /* 从 TCB 中获取任务使用的 CSA 链表头 */
    ulHeadCSA = ( **pxTCB ) & portCSA_FCX_MASK;
    
    /* 遍历 CSA 链表，找到链尾 */
    for(pulNextCSA = pxPortCsaToAddress( ulHeadCSA );
        ( pulNextCSA[ 0 ] & portCSA_FCX_MASK ) != 0;
        pulNextCSA = pxPortCsaToAddress( pulNextCSA[ 0 ] ) )
    {
        pulNextCSA[ 0 ] &= portCSA_FCX_MASK;
    }
    
    /* 将任务的 CSA 链表插入到空闲链表头部 */
    __disable();
    {
        ulFreeCSA = __mfcr( portCPU_FCX );
        pulNextCSA[ 0 ] = ulFreeCSA;      // 链尾指向当前空闲链表头
        __mtcr( portCPU_FCX, ulHeadCSA ); // 更新 FCX 指向回收的 CSA 链表头
    }
    __enable();
}
```

## 7. CSA 使用注意事项

### 7.1 CSA 耗尽检测

- 如果 CSA 分配失败，会触发 `svlcx` 陷阱
- LCX 寄存器指向倒数第三个 CSA，用于检测 CSA 是否即将耗尽

### 7.2 任务嵌套调用深度

- 每个任务初始分配 2 个 CSA（Upper + Lower）
- 如果任务中有函数调用，会动态从空闲链表分配更多 CSA
- 函数返回时，CSA 会自动释放回空闲链表

### 7.3 多核环境

- 每个核心有独立的 CSA 池（8KB，128 个 CSA）
- 每个核心的 FCX 寄存器独立管理自己的空闲 CSA 链表
- 任务只能使用其所在核心的 CSA 池

## 8. 调试建议

### 8.1 检查 CSA 使用情况

可以通过以下方式检查 CSA 使用情况：

1. **查看 FCX 寄存器**：了解当前空闲 CSA 数量
2. **查看 LCX 寄存器**：如果接近 LCX，说明 CSA 即将耗尽
3. **监控任务栈**：检查栈顶的 CSA 地址是否正确

### 8.2 常见问题

1. **CSA 耗尽**：
   - 症状：触发 `svlcx` 陷阱
   - 原因：任务嵌套调用过深，或 CSA 池配置过小
   - 解决：增加 `LCF_CSAx_SIZE` 或减少任务嵌套深度

2. **任务不执行**：
   - 检查任务创建时 CSA 分配是否成功
   - 检查任务栈中的 CSA 地址是否有效

## 9. CSA 分配机制总结

### 9.1 核心概念

**CSA 是共享资源，不是任务私有资源**：

1. **每个 CPU 核心有独立的 CSA 池**：
   - 每个核心有 8KB CSA 池（128 个 CSA）
   - 通过 FCX 寄存器管理的空闲链表

2. **任务不拥有固定的 CSA**：
   - 任务创建时：从全局池分配 2 个 CSA 用于初始化
   - 任务运行时：根据函数调用深度，动态从全局池分配/释放 CSA
   - 任务切换时：当前 CSA 链状态保存到任务栈
   - 任务删除时：任务使用的所有 CSA 回收到全局池

3. **CSA 的动态分配机制**：
   - 函数调用时：硬件自动从 FCX 指向的空闲链表分配 CSA
   - 函数返回时：硬件自动将 CSA 释放回空闲链表
   - 任务切换时：通过 PCXI 寄存器保存/恢复 CSA 链状态

### 9.2 关键区别

| 特性 | CPU 核心 | 任务 |
|------|---------|------|
| CSA 池 | 每个核心有独立的 8KB 池 | 无固定池 |
| CSA 分配 | 固定大小（128 个） | 动态分配（初始 2 个，运行时变化） |
| CSA 管理 | FCX 寄存器管理空闲链表 | 通过任务栈保存 CSA 链状态 |
| CSA 生命周期 | 系统启动时初始化，一直存在 | 任务创建时分配，删除时回收 |

### 9.3 实际运行示例

假设有 3 个任务在 CPU0 上运行：

```
CPU0 CSA 池（128 个 CSA，8KB）
├── 空闲 CSA 链表（FCX 指向）
│   ├── CSA #10
│   ├── CSA #11
│   └── ...
│
├── Task1 使用的 CSA
│   ├── Lower CSA #1  ← 任务创建时分配
│   └── Upper CSA #2  ← 任务创建时分配
│   └── CSA #5        ← 函数调用时动态分配
│
├── Task2 使用的 CSA
│   ├── Lower CSA #3  ← 任务创建时分配
│   └── Upper CSA #4  ← 任务创建时分配
│
└── Task3 使用的 CSA
    ├── Lower CSA #6  ← 任务创建时分配
    └── Upper CSA #7  ← 任务创建时分配
    └── CSA #8        ← 函数调用时动态分配
    └── CSA #9        ← 函数调用时动态分配
```

**要点**：
- 所有任务共享同一个 CPU 核心的 CSA 池
- 任务使用的 CSA 数量是动态变化的
- 任务切换时，CSA 链状态保存在任务栈中，CSA 本身仍在全局池中

---

**文档生成时间**：2024-12-19

