---
name: wdm-driver-expert
description: Windows WDM 驱动开发专家助手，提供完整的 WDM 开发指导、代码模板、问题排查和架构设计
---

# WDM 驱动开发专家 Skill

专注于 Windows WDM 驱动开发专家助手，为 WDM (Windows Driver Model）驱动开发提供专业指导、架构设计、代码模板、问题排查。

## When to use

当用户需要以下帮助时使用：
- WDM 驱动开发架构设计
- 需要 WDM 驱动代码模板
- WDM 驱动调试与问题排查
- IRP 处理流程设计
- 设备对象和设备扩展设计
- WDM 驱动分层架构指导
- WDM 即插即用 (PNP) 处理实现
- 电源管理实现指导
- WDM 驱动兼容性问题分析

## WDM 基础架构知识

### WDM 驱动分层结构

```
上层：总线驱动 → 功能驱动 → 过滤驱动

```

**各层职责：**

1. **总线驱动 (Bus Driver)**
   - 枚举总线上的设备
   - 创建物理设备对象 (PDO)
   - 处理总线级 PNP 请求
   - 例如：PCI 总线驱动、USB 总线驱动

2. **功能驱动 (Function Driver)**
   - 实现设备主要功能
   - 实现设备的 I/O 处理
   - 创建功能设备对象 (FDO)

3. **过滤驱动 (Filter Driver)**
   - 在功能驱动之上或之间
   - 拦截或修改 I/O 请求
   - 不改变设备原有功能

### 核心数据结构

```c
// 驱动入口
NTSTATUS DriverEntry(_In_ PDRIVER_OBJECT DriverObject, _In_ PUNICODE_STRING RegistryPath);

// 设备扩展 - 存储设备特定数据
typedef struct _DEVICE_EXTENSION {
    PDEVICE_OBJECT DeviceObject;     // 我们自己的设备对象
    PDEVICE_OBJECT LowerDevice;      // 下层设备对象
    ... // 自定义设备数据
} DEVICE_EXTENSION, *PDEVICE_EXTENSION;

// IRP 主派遣函数
NTSTATUS DispatchXxx(_In_ PDEVICE_OBJECT DeviceObject, _In_ PIRP Irp);

// Unload 例程
VOID DriverUnload(_In_ PDRIVER_OBJECT DriverObject);
```

## WDM 驱动必须实现的派遣函数

| IRP 主功能代码 | 必须实现 | 说明 |
|----------------|----------|------|
| IRP_MJ_CREATE | ✅ | 创建设备 |
| IRP_MJ_CLOSE | ✅ | 关闭设备 |
| IRP_MJ_READ | 按需 | 读操作 |
| IRP_MJ_DEVICE_CONTROL | 按需 | IOCTL |
| IRP_MJ_PNP | ✅ | 即插即用 |
| IRP_MJ_POWER | ✅ | 电源管理 |
| IRP_MJ_SYSTEM_CONTROL | WMI | 系统控制 |

## 代码模板

### 完整 WDM 驱动最小模板

```c
#include <ntddk.h>
#include <wdm.h>

// 设备扩展
typedef struct _DEVICE_EXTENSION {
    PDEVICE_OBJECT  DeviceObject;
    PDEVICE_OBJECT  LowerDeviceObject;
    BOOLEAN         Removed;
} DEVICE_EXTENSION, *PDEVICE_EXTENSION;

// 前置声明
DRIVER_INITIALIZE DriverEntry;
DRIVER_UNLOAD DriverUnload;
DRIVER_DISPATCH DispatchCreate;
DRIVER_DISPATCH DispatchClose;
DRIVER_DISPATCH DispatchPnp;
DRIVER_DISPATCH DispatchPower;
DRIVER_DISPATCH DispatchDeviceControl;

//===========================================================================
// 驱动入口
//===========================================================================
NTSTATUS
DriverEntry(
    _In_ PDRIVER_OBJECT  DriverObject,
    _In_ PUNICODE_STRING RegistryPath
)
{
    UNREFERENCED_PARAMETER(RegistryPath);

    // 设置派遣函数
    DriverObject->DriverUnload = DriverUnload;
    DriverObject->MajorFunction[IRP_MJ_CREATE] = DispatchCreate;
    DriverObject->MajorFunction[IRP_MJ_CLOSE] = DispatchClose;
    DriverObject->MajorFunction[IRP_MJ_PNP] = DispatchPnp;
    DriverObject->MajorFunction[IRP_MJ_POWER] = DispatchPower;
    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = DispatchDeviceControl;

    DbgPrint("WDM Driver Loaded\n");
    return STATUS_SUCCESS;
}

//===========================================================================
// 创建设备 - 一般在 AddDevice 中完成
//===========================================================================
NTSTATUS
AddDevice(
    _In_ PDRIVER_OBJECT DriverObject,
    _In_ PDEVICE_OBJECT PhysicalDeviceObject
)
{
    NTSTATUS status;
    PDEVICE_OBJECT deviceObject;
    PDEVICE_EXTENSION deviceExtension;

    // 创建功能设备对象 (FDO)
    status = IoCreateDevice(
        DriverObject,
        sizeof(DEVICE_EXTENSION),
        NULL,
        FILE_DEVICE_UNKNOWN,
        0,
        FALSE,
        &deviceObject
    );

    if (!NT_SUCCESS(status)) {
        return status;
    }

    deviceExtension = (PDEVICE_EXTENSION)deviceObject->DeviceExtension;
    deviceExtension->DeviceObject = deviceObject;
    deviceExtension->LowerDeviceObject = PhysicalDeviceObject;

    // 附加到设备栈
    deviceExtension->LowerDeviceObject = IoAttachDeviceToDeviceStack(
        deviceObject,
        PhysicalDeviceObject
    );

    // 设置标志
    if (deviceExtension->LowerDeviceObject->Flags & DO_BUFFERED_IO) {
        deviceObject->Flags |= DO_BUFFERED_IO;
    }
    if (deviceExtension->LowerDeviceObject->Flags & DO_DIRECT_IO) {
        deviceObject->Flags |= DO_DIRECT_IO;
    }

    deviceObject->DeviceType = deviceExtension->LowerDeviceObject->DeviceType;

    deviceObject->Characteristics = 0;

    return STATUS_SUCCESS;
}

//===========================================================================
// 完成 IRP 通用方法
//===========================================================================
NTSTATUS
CompleteIrp(
    _In_ PIRP Irp,
    _In_ NTSTATUS status,
    _In_ ULONG_PTR information
)
{
    Irp->IoStatus.Status = status;
    Irp->IoStatus.Information = information;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return status;
}

//===========================================================================
// 创建派遣
//===========================================================================
NTSTATUS
DispatchCreate(
    _In_ PDEVICE_OBJECT DeviceObject,
    _In_ PIRP Irp
)
{
    UNREFERENCED_PARAMETER(DeviceObject);
    return CompleteIrp(Irp, STATUS_SUCCESS, 0);
}

//===========================================================================
// 关闭派遣
//===========================================================================
NTSTATUS
DispatchClose(
    _In_ PDEVICE_OBJECT DeviceObject,
    _In_ PIRP Irp
)
{
    UNREFERENCED_PARAMETER(DeviceObject);
    return CompleteIrp(Irp, STATUS_SUCCESS, 0);
}

//===========================================================================
// PNP 派遣 - 默认传递到下一层
//===========================================================================
NTSTATUS
DispatchPnp(
    _In_ PDEVICE_OBJECT DeviceObject,
    _In_ PIRP Irp
)
{
    PDEVICE_EXTENSION deviceExtension = (PDEVICE_EXTENSION)DeviceObject->DeviceExtension;
    return IoForwardIrpSynchronous(DeviceObject, deviceExtension->LowerDeviceObject, Irp);
}

//===========================================================================
// 电源派遣 - 默认传递到下一层
//===========================================================================
NTSTATUS
DispatchPower(
    _In_ PDEVICE_OBJECT DeviceObject,
    _In_ PIRP Irp
)
{
    PDEVICE_EXTENSION deviceExtension = (PDEVICE_EXTENSION)DeviceObject->DeviceExtension;
    PoStartNextPowerIrp(Irp);
    return IoCallDriver(deviceExtension->LowerDeviceObject, Irp);
}

//===========================================================================
// IOCTL 处理
//===========================================================================
NTSTATUS
DispatchDeviceControl(
    _In_ PDEVICE_OBJECT DeviceObject,
    _In_ PIRP Irp
)
{
    PIO_STACK_LOCATION irpStack = IoGetCurrentIrpStackLocation(Irp);
    ULONG ioControlCode = irpStack->Parameters.DeviceIoControl.IoControlCode;

    switch (ioControlCode) {
    // 在这里处理你的 IOCTL
    default:
        break;
    }

    return CompleteIrp(Irp, STATUS_SUCCESS, 0);
}

//===========================================================================
// 驱动卸载
//===========================================================================
VOID
DriverUnload(
    _In_ PDRIVER_OBJECT DriverObject
)
{
    UNREFERENCED_PARAMETER(DriverObject);
    DbgPrint("WDM Driver Unloaded\n");
}
```

## WDM IRP 处理流程

### 转发 IRP 到下层驱动

两种方式：

**1. 同步转发（推荐简单情况）:**
```c
// 将 IRP 转发给下层驱动，等待完成
IoForwardIrpSynchronous(DeviceObject, lowerDevice, Irp);
```

**2. 异步转发:**
```c
// 设置完成例程后转发
IoCopyCurrentIrpStackLocationToNext(Irp);
IoSetCompletionRoutine(Irp, MyCompletionRoutine, context, TRUE, TRUE, TRUE);
return IoCallDriver(lowerDevice, Irp);
```

### PNP IRP 处理要点

| PNP Minor Function | 处理要点 |
|---------------------|----------|
| IRP_MN_START_DEVICE | 分配资源，初始化硬件 |
| IRP_MN_REMOVE_DEVICE | 删除设备，释放所有资源 |
| IRP_MN_STOP_DEVICE | 停止设备，释放资源 |
| IRP_MN_QUERY_STOP_DEVICE | 查询是否可以停止 |
| IRP_MN_SURPRISE_REMOVAL | 意外移除处理 |

### 电源 IRP 处理要点

- 必须调用 `PoStartNextPowerIrp`
- 转发到下级
- 设置 S0 之前设置系统到工作状态

## 常见问题排查

### BugCheck (0x50 PAGE_FAULT_IN_NONPAGED_AREA

**原因:**
- 在 DISPATCH_LEVEL 访问分页内存
- 访问已释放内存
- 错误的指针

**排查方法:**
```
1. 使用 !drvobj 检查设备对象
2. 检查调用者当前 IRQL
3. 使用 !process 查看当前进程
4. 检查内存分配类型

### BugCheck (0x7F UNEXPECTED_KERNEL_MODE_TRAP)

**常见原因:**
- 栈溢出
- 双重释放
- 调用了不匹配的自旋锁问题

### 内存泄漏排查

1. 使用 PoolMon 工具:
```
poolmon.exe - 查看 pool tags
pool tag 在 `!poolused 在内核调试中

## WDM vs WDF 区别

| 特性 | WDM | WDF (KMDF) |
|------|-----|-----------|
| 对象生命周期 | 手动管理 | 框架管理 |
| PNP/电源 | 全手动处理 | 框架处理大部分 |
| 错误处理 | 全手动 | 框架帮你处理 |
| 代码量 | 多 | 少 |
| 学习难度 | 高 | 较低 |
| 推荐度 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

**什么时候还需要 WDM:**
- 某些底层驱动（总线驱动、过滤驱动）
- 需要完全控制驱动流程
- legacy 设备兼容要求
- 对 Windows 版本兼容性要求很高

## 调试技巧

### 动态加载调试命令（WinDbg:

```k
!drvobj <drivername> 查看驱动对象
!devstack <device> 查看设备栈
!pool 查看内存使用
!pcr 查看处理器控制区域
```

### 追踪 IRP 跟踪
```
!irpfind 查找所有未完成 IRP
!irp 查看 IRP 详情
```

## 最佳实践

1. **总是在正确释放资源:** 总是检查返回值检查
2. 总是在 `AddDevice` 正确创建设备对象
3. 在 `IRP_MN_REMOVE_DEVICE` 时释放全部资源
4. 正确设置 I/O 方式（Buffered IO vs Direct IO）
5. 不要在不正确的 IRQL 访问分页内存
6. 使用 `ExAllocatePoolWithTag，不要使用已废弃的 `ExAllocatePool`
7. 总是配对对调用 IoCompleteRequest，不要泄漏 IRP

## IOCTL 定义规范

```c
// 自定义 IOCTL 定义方式:

#define IOCTL_MYDRV_IOCTL_FUNC \
    CTL_CODE(FILE_DEVICE_UNKNOWN, 0x800, METHOD_BUFFERED, FILE_ANY_ACCESS)

```

- 函数码从 0x800 开始，避免和系统冲突
- 根据使用场景选择 METHOD_BUFFERED / METHOD_IN_DIRECT / METHOD_OUT_DIRECT / METHOD_NEITHER

## 帮助能力

当用户问你：
- "给我一个完整的 WDM 驱动模板 → 提供最小可编译模板
- "帮我分析这个 WDM bug → 按检查常见问题排查流程
- WDM 架构设计 → 给出分层架构建议
- WDM 代码审查 → 结合 windows-driver-codereview 配合使用
