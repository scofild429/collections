---
craft_id: FE960577-F583-4F1A-A152-364F03E6D5DE
craft_source: "craftdocs://open?spaceId=f0e27734-d8b8-47ce-be9d-9b35def0cb70&blockId=FE960577-F583-4F1A-A152-364F03E6D5DE"
imported: 2026-09-27
---

# Mac

## 修复 iPhone → Mac Universal Clipboard 单向失效

### 现象与已完成的排查

原笔记记录：iPhone 与 Mac mini 使用同一 Apple Account（原称 Apple ID）；Mac → iPhone 通用剪贴板正常，但 iPhone → Mac 无法粘贴。Handoff 功能正常，相关进程存在，Mac 本地 `pbcopy` / `pbpaste` 正常。

这些结果说明本地剪贴板可用，但不能单独证明跨设备传输链路完全正常。

### 先检查官方前提

两台设备应支持 Universal Clipboard、彼此靠近、登录同一 Apple Account，并开启 Wi-Fi、蓝牙和 Handoff。复制内容只会暂时保留供跨设备粘贴；在另一台设备复制新内容也会替换它。参见 [Apple 通用剪贴板说明](https://support.apple.com/en-us/102430)。

### 此机器上有效的临时处理

原笔记记录，执行以下命令后恢复正常：

```bash
killall useractivityd
killall sharingd
```

这会请求终止当前用户可操作的同名进程，可能暂时中断接力或共享功能。原记录中相关服务自动重新启动，无需 `sudo` 或重启整机；具体行为取决于 macOS 版本和服务状态。

这是一条个人故障处理记录，不是 Apple 保证有效的通用修复步骤。若提示没有匹配进程，不代表必须使用 `sudo`。重新复制一段新文本并测试两个方向；若仍失败，重新检查官方前提，并尝试正常重启设备。
