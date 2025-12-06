# Google Earth Flatfile 解密指南

## 概述

本文档记录了对 Google Earth Pro `flatfile` 端点加密数据的逆向分析结果，以及获取历史影像时间轴数据的替代方案。

## 加密机制分析

### Flatfile 端点
```
URL: https://khmdb.google.com/flatfile?db=tm&f1-{octant}-i.{epoch}-{hash}
```

### 加密参数

| 参数 | 值 |
|------|-----|
| **算法** | AES-128-GCM |
| **数据格式** | `[IV (12字节)] [Tag (16字节)] [密文]` |
| **密钥派生** | OpenSSL `EVP_BytesToKey` |
| **哈希算法** | SHA256 |
| **迭代次数** | 5 |
| **Salt** | NULL (无盐值) |
| **密码来源** | Windows 凭据管理器 (QKeychain)，密钥名 `E97YiBb33i` |

### 关键函数地址 (googleearth_pro.dll)

| 函数 | 偏移地址 | 说明 |
|------|---------|------|
| 解密入口 | `+0x7E7620` | `sub_7E7620` - 主解密函数 |
| EVP_BytesToKey | `+0x7E7714` | 密钥派生调用 |
| EVP_DecryptInit_ex | `+0x7E791C` | AES-GCM 初始化 |
| 密码获取 | `+0x7E4AB0` | QKeychain 调用 |

---

## 解密方法

### 方法 1：从 Windows 系统提取密钥

**使用 NirSoft CredentialsFileView：**
1. 下载 [CredentialsFileView](https://www.nirsoft.net/utils/credentials_file_view.html)
2. 填写 `C:\Users\{用户名}\AppData\Local\Microsoft\Vault`
3. 输入 Windows 登录密码
4. 搜索包含 `GoogleEarth` 或 `E97YiBb33i` 的条目

### 方法 2：x64dbg 动态调试

1. 运行 `x32dbg.exe`（Google Earth 是32位程序）
2. 附加到 `googleearth.exe`
3. 按 `Alt+E` 找到 `googleearth_pro.dll` 基址
4. 计算断点地址：`基址 + 0x7E791C`
5. `Ctrl+G` 跳转，`F2` 设置断点
6. `F9` 运行，触发时间轴请求
7. 断点命中后查看栈上第5个参数（Key 指针）

### 方法 3：Frida Hook

```javascript
// frida -p <pid> -l hook.js
Interceptor.attach(Module.findExportByName("libeay32.dll", "EVP_DecryptInit_ex"), {
    onEnter: function(args) {
        console.log("Key:", hexdump(args[3], {length: 16}));
        console.log("IV:", hexdump(args[4], {length: 12}));
    }
});
```

---

## 替代方案：BulkMetadata API

无需解密即可获取时间轴数据！

### 端点
```
https://kh.google.com/rt/tm/earth/BulkMetadata/pb=!1m2!1s{node_key}!2u{epoch}!4b1
```

### 使用脚本
```bash
python3 get_timeline.py --lat 35.6586 --lon 139.7454
```

### 返回数据
- 528 个节点的元数据
- 28+ 个历史影像 epoch
- 每个 epoch 可转换为日期

---

## 文件说明

| 文件 | 说明 |
|------|------|
| `1.py` | Flatfile 获取和解密尝试脚本 |
| `get_timeline.py` | BulkMetadata API 时间轴提取 |
| `timestamp_converter.py` | Epoch 到日期转换工具 |
| `download_history.py` | 历史影像瓦片下载 |
| `rocktree_pb2.py` | Protobuf 定义 |

---

## Epoch 日期格式

### 长格式 (11位)
```
10348872021 = 2021-04-07
格式: 103 + 月 + (880+日) + 年
```

### 短格式 (7位)
```
1035370 ≈ 2022-03-10
需要查表或进一步分析
```

---

## 下一步

1. ✅ 使用 `get_timeline.py` 获取时间轴数据（无需解密）
2. ⏳ 提取 QKeychain 中的实际密码（需要动态调试）
3. ⏳ 完善 epoch 到精确日期的转换算法

## 参考

- [OpenSSL EVP_BytesToKey](https://www.openssl.org/docs/man1.1.1/man3/EVP_BytesToKey.html)
- [Qt QKeychain](https://github.com/nickhudkins/qtkeychain)
- [Google Earth Rocktree Protocol](https://github.com/nickhudkins/earth-reverse-engineering)
