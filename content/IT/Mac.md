---
craft_id: FE960577-F583-4F1A-A152-364F03E6D5DE
craft_source: "craftdocs://open?spaceId=f0e27734-d8b8-47ce-be9d-9b35def0cb70&blockId=FE960577-F583-4F1A-A152-364F03E6D5DE"
imported: 2026-09-27
---

# Mac

## 修复 iPhone → Mac Universal Clipboard 单向失效

**现象**：iPhone 与 Mac mini 使用同一 Apple ID。Mac → iPhone 通用剪贴板正常，但 iPhone → Mac 无法粘贴。

**排查**：Handoff 正常，相关进程均在运行，Mac 本地 `pbcopy` / `pbpaste` 正常。

**解决**：在 Mac Terminal 中执行以下命令后恢复正常：

```bash
killall useractivityd
killall sharingd
```

macOS 会自动重启这两个进程，无需 `sudo`，也无需重启整机。
