---
name: windows-driver-codereview
description: 专业的 Windows 驱动开发代码审查工具，包含 WDM、WDF、KMDF、UMDF 驱动开发的最佳实践和安全检查清单
---

# Windows 驱动开发 CodeReview Skill

专门用于 Windows 驱动开发代码审查的技能，涵盖内存管理、同步、IRP 处理、安全、性能等多个方面的检查点。

## When to use

当用户需要对以下场景进行代码审查时使用：
- Windows 内核驱动开发 (KMDF/WDM)
- 用户模式驱动开发 (UMDF)
- Windows 框架驱动 (WDF)
- 设备驱动程序代码审查
- 驱动相关的 PR/MR 审查
- Windows 内核模式代码评审

## 审查检查清单

### 1. 内存管理检查

#### ✅ 必须检查：
- [ ] 所有非分页内存分配是否在正确的 IRQL 级别使用 (`<= DISPATCH_LEVEL`)
- [ ] `ExAllocatePoolWithTag` 使用是否正确，必须指定标签（不允许使用 0）
- [ ] 内存分配后是否检查返回值，处理分配失败情况
- [ ] 分配的内存在不再使用时是否正确释放
- [ ] 栈上分配内存不超过页面大小（通常 4KB）
- [ ] 避免在分页内存中存储敏感信息（如密码、密钥）
- [ ] `MmGetSystemAddressForMdlSafe` 调用后必须检查返回值
- [ ] 是否正确处理了 MDL（内存描述符列表）的申请和释放

#### 🚫 常见错误：
- 在 `DISPATCH_LEVEL` 级别使用分页内存分配
- 忘记检查内存分配返回值导致空指针引用
- 内存泄漏（分配后未释放）
- 使用 `ExAllocatePool` 已废弃 API，应使用带 Tag 的版本

---

### 2. IRQL 与中断处理检查

#### ✅ 必须检查：
- [ ] 当前 IRQL 是否允许执行该操作
  - `>= DISPATCH_LEVEL` 不能访问分页内存
  - `>= HIGH_LEVEL` 只能调用非常有限的内核 API
- [ ] `KeRaiseIrql`/`KeLowerIrql` 是否成对出现
- [ ] 自旋锁获取/释放是否正确配对，且 IRQL 级别正确
- [ ] 中断服务例程 (ISR) 中是否只调用了允许的内核 API
- [ ] DPC 例程中是否正确访问内存（分页内存禁止）
- [ ] `KeAcquireSpinLockAtDpcLevel`/`KeReleaseSpinLockFromDpcLevel` 使用是否正确

#### 🚫 常见错误：
- 在 DPC 中访问分页内存导致 bugcheck
- IRQL 不匹配导致的栈损坏
- 忘记降低 IRQL 导致死锁
- 在错误 IRQL 级别调用了分页 API

---

### 3. 同步与并发检查

#### ✅ 必须检查：
- [ ] 共享数据访问是否都有正确的同步保护
- [ ] 自旋锁、互斥量、快速互斥量的使用场景是否正确
  - 短时间锁：自旋锁（适用于 IRQL <= DISPATCH_LEVEL）
  - 长时间锁：互斥量（允许阻塞）
- [ ] 是否存在可能的死锁情况（锁获取顺序是否一致）
- [ ] 引用计数是否正确（增加/减少配对）
- [ ] `InterlockedXxx` 函数使用是否正确，类型匹配
- [ ] 完成例程中的同步处理是否正确

#### 🚫 常见错误：
- 共享数据未同步导致竞态条件
- 在持有自旋锁时执行阻塞操作
- 锁顺序反转导致死锁
- 引用计数错误导致对象提前释放或泄漏

---

### 4. IRP 处理检查

#### ✅ 必须检查：
- [ ] 所有 IRP 最终是否都调用 `IoCompleteRequest`
- [ ] `IoMarkIrpPending` 返回 `STATUS_PENDING` 是否正确配对
- [ ] IRP 队列管理是否正确，避免队列溢出
- [ ] 取消例程是否正确设置和处理
- [ ] I/O 栈位置 (`IoGetCurrentIrpStackLocation`) 获取是否正确
- [ ] 设备扩展指针是否正确验证
- [ ] 缓冲 I/O 和直接 I/O 的处理是否符合设备类型约定

#### 🚫 常见错误：
- IRP 泄漏（未调用 `IoCompleteRequest`）
- 既不完成也不挂起 IRP（"pend-complete" 错误）
- 错误地处理了取消 IRP
- 使用了错误的 I/O 栈位置

---

### 5. WDF 特定检查

#### ✅ 必须检查：
- [ ] 对象生命周期管理是否正确
  - 父对象关系设置是否合理
  - 析构回调/清理回调是否正确清理资源
- [ ] 回调函数中是否正确处理上下文
  - `WdfObjectGetTypedContext` 使用是否正确
  - 结构签名验证是否存在（推荐）
- [ ] 内存类型使用是否正确
  - WDF 对象内存是否正确分配
  - `WdfMemoryCreate` 使用后是否正确释放
- [ ] 队列配置是否正确
  - 并行/顺序/手动队列选择是否正确
  - 请求完成是否正确调用 `WdfRequestComplete`
- [ ] DMA 操作是否符合 WDF 约定
- [ ] 中断对象配置是否正确

#### 🚫 常见错误：
- 对象上下文类型转换错误
- 在错误的回调中访问设备对象
- 忘记设置父对象导致对象泄漏
- 错误地手动删除了对象（应该依赖父对象层级管理）

---

### 6. 安全性检查

#### ✅ 必须检查：
- [ ] 所有来自用户模式的输入是否都进行了验证
  - 缓冲区长度验证
  - 指针验证
  - IOCTL 代码检查
- [ ] 是否正确使用了 `ProbeForRead`/`ProbeForWrite`
- [ ] 内存拷贝是否检查了边界（溢出保护）
- [ ] 权限检查是否到位
  - 用户请求是否验证了调用者权限
  - 特权操作是否检查了权限
- [ ] 加密密钥、凭据等敏感信息是否正确处理（不留在可分页内存）
- [ ] 是否禁用了不必要的 IOCTL 处理（特别是 METHOD_OUT_DIRECT）
- [ ] 字符串操作是否安全（避免 off-by-one 错误）

#### 🚫 常见漏洞：
- 缓冲区溢出
- 未验证用户输入导致内核信息泄露
- 空指针解引用
- 整数溢出导致内存分配错误
- 信息泄露（通过未初始化内存）

---

### 7. 错误处理检查

#### ✅ 必须检查：
- [ ] 所有内核 API 返回值是否都检查（特别是可能失败的调用）
- [ ] 错误路径中资源是否正确释放（避免泄漏）
- [ ] goto 错误退出模式是否正确清理
- [ ] `STATUS_XXX` 返回码是否使用正确
  - 成功：`STATUS_SUCCESS`
  - 错误：使用合适的 NTSTATUS 代码
  - 不应该将用户态错误码直接返回
- [ ] 设备启动失败时是否正确清理已分配资源

#### 🚫 常见错误：
- 错误路径中遗漏了资源释放导致泄漏
- 错误码使用不正确（如用 0 表示失败）
- 忽略了 `MmGetSystemAddressForMdlSafe` 等可能失败的 API 返回值

---

### 8. 性能检查

#### ✅ 必须检查：
- [ ] 是否持有锁的时间过长
- [ ] 内存分配是否在热路径中（影响性能）
- [ ] I/O 请求处理是否会导致过多的上下文切换
- [ ] 大内存分配是否适当（避免碎片化）
- [ ] 中断禁用时间是否过长
- [ ] DPC 例程执行时间是否过长

#### 💡 优化建议：
- 考虑使用 lookaside list 管理频繁分配释放的对象
- 避免在内核路径中做不必要的计算
- 考虑使用散列/批处理减少锁竞争

---

### 9. 代码规范与最佳实践

#### ✅ 必须检查：
- [ ] 驱动入口函数是否正确命名（`DriverEntry`）
- [ ] 所有函数都有合适的注释（特别是复杂逻辑）
- [ ] 魔术数字是否都定义为常量
- [ ] 使用了正确的类型（`PVOID`, `ULONG`, `NTSTATUS` 等）
- [ ] 不使用匈牙利命名法（Windows 内核开发约定）
- [ ] 避免了未初始化的局部变量
- [ ] 不使用 C++ 异常，驱动开发中禁止异常
- [ ] 全局变量使用是否合理（尽量少用）

---

### 10. PNP 和电源管理检查

#### ✅ 必须检查：
- [ ] PNP 状态转换处理是否正确
- [ ] 设备移除时所有资源是否正确释放
- [ ] 电源状态转换处理是否正确（S0 -> Sx -> S0）
- [ ] 唤醒信号处理是否正确
- [ ] 设备停止/重启是否正确处理挂起的 IRP

---

### 11. DMA 和硬件操作检查

#### ✅ 必须检查：
- [ ] DMA 地址转换是否正确（物理地址 <-> 虚拟地址）
- [ ] 缓存一致性处理是否正确
- [ ] 寄存器访问是否使用正确的宏（`READ_REGISTER_XXX`/`WRITE_REGISTER_XXX`）
- [ ] 端口访问是否正确（`READ_PORT_XXX`/`WRITE_PORT_XXX`）
- [ ] 内存映射 IO 区域访问是否正确保护
- [ ] DMA 完成后是否正确释放资源

---

## 使用方法

### 完整审查流程：

1. **接收代码片段或完整文件**
2. **按上述检查清单逐项检查**
3. **分类记录发现的问题**：
   - 🔴 **严重问题**：会导致系统崩溃、安全漏洞、内存泄漏
   - 🟡 **警告**：不符合最佳实践，可能导致问题，需要优化
   - 🟢 **建议**：代码风格、性能优化建议
4. **输出审查报告**：
   - 按问题严重程度排序
   - 每个问题说明位置、原因、修复建议
   - 提供修复后的代码示例

### 快速审查：

如果用户只需要快速检查，重点关注：
1. 内存管理和 IRQL
2. 安全性输入验证
3. 错误处理和资源释放
4. IRP 完整性

## 输出格式示例

```
## Windows 驱动代码审查报告

### 🔴 严重问题 (需要立即修复)

1. **文件: driver.c 行号: 123**
   - 问题: 在 `DISPATCH_LEVEL` 级别访问分页内存
   - 影响: 会导致系统 bugcheck (PAGE_FAULT_IN_NONPAGED_AREA)
   - 修复: 将该变量分配为非分页内存，或移动到 lower IRQL 访问

```c
// 修复前:
PVOID pagedBuffer = ExAllocatePool(PagedPool, size); // ❌
KeAcquireSpinLock(&lock, &oldIrql);
memcpy(pagedBuffer, src, size); // 在 DISPATCH_LEVEL 访问分页内存

// 修复后:
PVOID nonPagedBuffer = ExAllocatePoolWithTag(NonPagedPool, size, DRV_TAG); // ✅
if (!nonPagedBuffer) { return STATUS_INSUFFICIENT_RESOURCES; }
KeAcquireSpinLock(&lock, &oldIrql);
memcpy(nonPagedBuffer, src, size);
```

### 🟡 警告 (需要关注)

1. **文件: queue.c 行号: 45**
   - 问题: 忽略了 `IoInsertTailList` 返回值未检查（虽然很少失败，但规范要求检查）
   - 建议: 添加返回值检查和错误处理

### 🟢 优化建议

1. 考虑使用 `ExInitializeLookasideList` 管理频繁分配的请求对象，减少内存碎片化
2. ...

### ✅ 检查通过

- [x] 所有 IRP 都正确调用 IoCompleteRequest
- [x] 所有内存分配都检查了返回值
- [x] WDF 对象生命周期管理正确
```

## 参考资料

- [Windows Driver Documentation](https://learn.microsoft.com/en-us/windows-hardware/drivers/)
- [Windows Driver Frameworks (WDF)](https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/)
- [Windows Kernel Programming Best Practices](https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/windows-kernel-mode-driver-framework-design-guide)
- [Security Best Practices for Windows Drivers](https://learn.microsoft.com/en-us/windows-hardware/drivers/security/security-best-practices-for-drivers)
