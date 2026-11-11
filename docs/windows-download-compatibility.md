# Windows 下载兼容修复 / Windows download compatibility

## 中文

本修复基于上游 `VallierDev/uu-remote-linux` 的 v0.2.57（`fe052cdca7ad90d1ab7c27d031036b1071869070`），解决官方 Windows 客户端从 Linux 被控端下载文件时，请求缺少路径或传输卡住的问题。相关上游问题：[VallierDev/uu-remote-linux#208](https://github.com/VallierDev/uu-remote-linux/issues/208)。

### 根因与修改

1. **路径分隔符**：Windows 4.46.0.4716 的 `file_transfer_utils.cpp` 使用 `QString::lastIndexOf('\\')` 拆分所选文件路径。纯正斜杠路径导致任务中的目录和文件名为空，服务端只收到 `send_request{1 id}`。目录列表仅对 Windows 控制端转换路径分隔符；实际 Linux 路径仍由现有 `normalize()` 恢复。
2. **任务路由**：Windows `FileTransferTaskManager::onRequest` 按 `RpcRequest.header.request_id` 查找任务。Windows 会丢弃使用独立请求计数器的主机请求。因此仅对 Windows 使用任务编号作为请求头编号，覆盖文件询问、数据块和完成通知。
3. **文件块顺序**：Windows `FileReceivingTask::onFileBlock` 按到达顺序追加数据，不按块编号定位写入。按文件读取顺序同步入队，然后并发等待确认，保留原有 16 块传输窗口。

代码中的协议注释记录了对应的官方客户端函数和地址。未改变协议能力声明，也未通过提高被控端版本号绕过功能检查。

### 验证记录（2026-11-11）

- Linux：Ubuntu 24.04.4 LTS，ARM64，`UU_DESKTOP_BACKEND=none`。
- 控制端：官方 Windows UU 远程 4.46.0.4716。
- `cargo test --release --locked -- --test-threads=1`：365 通过，0 失败，11 跳过。
- `cargo build --release --locked --bin uuremote-server`：成功。
- 新增回归测试覆盖 Windows 路径拆分、中文及空格文件名、压缩和内联目录列表一致性，以及多线程下多个窗口的数据顺序和逆序确认。
- Windows 图形客户端实测下载 42 字节文本和 1,048,576 字节二进制文件，两者接收文件的 SHA-256 均与 Linux 源文件一致。

本次实机验收覆盖上述版本的单文件下载。Windows 上传、目录下载、断点续传和其他客户端版本尚未纳入本次实机验收，不能据此推断全部兼容。

### 获取修复

本 fork 的 `main` 包含修复；上游 v0.2.57 发布包不包含此补丁。请按 [README 的构建说明](../README.md) 安装依赖，再从本仓库构建：

```sh
git clone https://github.com/jyc2023-oss/uu-remote-linux.git
cd uu-remote-linux
cargo build --release --locked --bin uuremote-server
```

编译产物为 `target/release/uuremote-server`。安装、登录和服务配置沿用 README；已有运行实例请保留原程序并按原部署方式替换，再重启服务。

## English

This fix is based on upstream v0.2.57 (`fe052cdca7ad90d1ab7c27d031036b1071869070`) and addresses missing-path requests and stalled downloads from Linux to the official Windows viewer. Related upstream report: [VallierDev/uu-remote-linux#208](https://github.com/VallierDev/uu-remote-linux/issues/208).

### Changes and evidence

- Windows 4.46.0.4716 splits selected paths on backslashes only. Directory listings now use Windows separators for Windows peers, while the existing normalization restores real Linux filesystem paths.
- Its task manager routes host requests by the RPC header request ID. Windows peers therefore use the transfer task ID for file asks, blocks and completion messages; other peers retain the existing request counter.
- Its receiving task appends blocks in arrival order. Blocks are now enqueued in file order before spawning acknowledgement waiters, preserving the 16-block window.

Source comments identify the official viewer functions and addresses supporting these changes. Feature declarations and the advertised host version are unchanged.

### Validation on 2026-11-11

On headless Ubuntu 24.04.4 LTS ARM64, `cargo test --release --locked -- --test-threads=1` passed with **365 passed, 0 failed, 11 ignored**, and the release host build succeeded. New regression tests cover path parsing, Unicode and spaces, compressed/inline listings, ordered multi-window transmission and reversed acknowledgements.

The official Windows viewer 4.46.0.4716 successfully downloaded a 42-byte text file and a 1,048,576-byte binary file. Both downloaded SHA-256 digests matched their Linux sources. Windows uploads, directory downloads, resume behavior and other client versions were not part of this live acceptance.

Build this fork's `main` using the commands above after installing the dependencies in [README.en.md](../README.en.md). The upstream v0.2.57 binaries do not contain this patch. Existing installation, login and service instructions still apply; retain the original executable when updating an installation.
