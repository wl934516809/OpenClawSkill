# Windows 驱动开发 CodeReview Skill

适用于 OpenClaw/Agents 的专业 Windows 驱动开发代码审查技能。

## 功能特性

提供完整的 Windows 内核驱动代码检查清单，涵盖：

- ✅ 内存管理检查（非分页/分页内存使用、IRQL 约束）
- ✅ IRQL 与中断处理检查
- ✅ 同步并发与死锁预防检查
- ✅ IRP 处理完整性检查
- ✅ WDF/KMDF/UMDF 特定规范检查
- ✅ 安全性检查（输入验证、缓冲区溢出保护）
- ✅ 错误处理与资源释放检查
- ✅ PNP/电源管理检查
- ✅ DMA 与硬件操作检查
- ✅ 性能优化建议

## 支持的驱动类型

- **KMDF** - Kernel-Mode Driver Framework
- **WDM** - Windows Driver Model
- **UMDF** - User-Mode Driver Framework
- **WDF** - Windows Driver Framework

## 安装

```bash
npx skills add https://github.com/wl934516809/windows-driver-codereview -s windows-driver-codereview -g
```

## 使用

在 Agent 中，当需要审查 Windows 驱动代码时，会自动激活此 Skill。

审查输出按照问题严重程度分类：

- 🔴 **严重问题** - 需要立即修复（会导致系统崩溃、安全漏洞）
- 🟡 **警告** - 不符合最佳实践，可能存在问题
- 🟢 **建议** - 代码风格、性能优化建议

## 检查清单示例

请查看 [SKILL.md](./SKILL.md) 了解完整的检查清单。

## 作者

https://github.com/wl934516809

## 许可证

MIT
