---
sidebar_position: 1
---

# 安装

Walrus 为 macOS（Intel 和 Apple CPU）和 Ubuntu 提供了预编译的walrus客户端二进制文件。

## 安装前提
保证安装了Sui Cli 并已配置钱包，调整网络到 testnet，并获取gas。
```shell
// 安装
brew install sui
// 设置测试网rpc
sui client new-env --alias testnet --rpc https://fullnode.testnet.sui.io:443
// 切换到测试网
sui client switch --env testnet
// 获取gas
sui client faucet
```

## 安装和配置
下载二进制文件:
- [Mac Apple Silicon下载](https://storage.googleapis.com/mysten-walrus-binaries/walrus-latest-macos-arm64)
- [Mac Intel下载](https://storage.googleapis.com/mysten-walrus-binaries/walrus-latest-macos-x86_64)
- [Ubuntu Intel下载](https://storage.googleapis.com/mysten-walrus-binaries/walrus-latest-ubuntu-x86_64)

手动配置CLI快捷使用：
```shell
// 把下载的二进制文件放到一个位置，建议.local/bin/下面
mv walrus ～/.local/bin/
// 打开配置文件
vim ~/.profile
// 查看是否有～/.local/bin/的目录配置, 没有添加上
export PATH="/Users/yourname/.local/bin:$PATH"
// 保存退出后source
source ~/.profile
// 查看
which walrus
```

配置Sui系统对象地址信息：
```shell
mkdir -p ~/.config/walrus
curl https://storage.googleapis.com/mysten-walrus-binaries/walrus-configs/client_config.yaml \
     -o ~/.config/walrus/client_config.yaml
```
内部其实只有一个Sui 上[系统对象](https://suiscan.xyz/testnet/object/0x6c957cf363ec968582f24e3e1a638c968cec1fa228999c560ec7925994906315)的 ID












