# Omnixie 网页烧录包 (ESP Web Tools)

用户在 Chrome/Edge 里打开网页、插上时钟、点一下就能刷固件。跨 Windows / Mac，无需装软件。

## 目录内容

| 文件 | 作用 |
|------|------|
| `index.html` | 烧录页面（中英双语，含 Install 按钮） |
| `manifest.json` | 告诉 ESP Web Tools 芯片型号(ESP8266)和固件写入地址(0x0)，指向 `firmware-v2.3.bin` |
| `firmware-v2.3.bin` | 编译好的固件 v2.3（**版本化文件名**，升版时换新名） |
| `latest.json` | App 内 OTA 用的版本清单（version/url/sha256/size） |

## 托管地址：docs.omnixie.com/omnixie/webflash/ (GitHub Pages)

`docs.omnixie.com` 是 **GitHub Pages**（DNS 指向 `omnixie.github.io`，自带 HTTPS，正好满足 Web Serial 要求）。托管 = 把这些文件放进为 `docs.omnixie.com` 服务的那个 GitHub 仓库、推送即可。

**步骤**：
1. 打开为 docs.omnixie.com 服务的仓库（就是绑了自定义域名 docs.omnixie.com、含 `assets/` 手册 PDF 的那个 Pages 仓库）。
2. 在仓库里建目录 `omnixie/webflash/`，把本目录的 5 个文件放进去。
3. （建议）在**仓库根目录**放一个空的 `.nojekyll` 文件，避免 Jekyll 处理静态/二进制文件。
4. `git commit` + `git push`。等约 1 分钟 GitHub Pages 重新构建。
5. 访问 `https://docs.omnixie.com/omnixie/webflash/` 验证。

所有文件必须放在**同一目录**（`index.html` 相对引用 `manifest.json`，manifest 引用 `firmware-v2.3.bin`）。

**缓存注意**：
- `firmware-v2.3.bin` 带版本号，升版换新名 → 永远是新 URL，不会发旧固件。
- `latest.json` / `manifest.json` 是"指针"，GitHub Pages(Fastly) 默认缓存约 10 分钟，升版后稍等即可（比 Cloudflare 的 4 小时好很多）。

**升级固件版本时**：编译出新 bin → 命名 `firmware-vX.Y.bin` 放进同目录 → 更新 `manifest.json` 的 `path` 和 `latest.json` 的 `version/url/sha256/size` → 推送。旧的 bin 可保留（老链接仍可用）。

## 用户使用步骤

1. 用 Chrome 或 Edge 打开托管好的网址（Safari/Firefox 不支持）。
2. USB-C 线连接时钟。
3. 点 Install，选名字带 `usbserial` / `CH340` 的串口。
4. 等待完成并自动重启。

Windows 若看不到串口，需装 WCH 的 CH340 驱动；macOS 新版一般免驱动。

## 关于是否清空设置

`manifest.json` 里 `"new_install_prompt_erase": false` —— 烧录只写程序区(0x0)，**保留** SPIFFS 里的 Wi-Fi/闹钟等设置，适合老用户升级。
若给全新设备想连带清空一切，改成 `true`（ESP Web Tools 会先整片擦除再烧）。

## 以后更新固件怎么重新生成 firmware.bin

在装好工具链的 Mac 上（arduino-cli + ESP8266 core 2.3.0 + 库：ArduinoJson v5、Timezone、Time、AES_Lib、SSDPDevice）：

```bash
# 把 esp8266/ 源码复制成同名 sketch 目录（arduino-cli 要求 .ino 与目录同名）
BUILD=/tmp/OmnixieBuild/Omnixiev1.1.5
mkdir -p "$BUILD" && cp esp8266/Omnixiev1.1.5.ino esp8266/myWebServer.cpp esp8266/myWebServer.h esp8266/htmlEmbed.h "$BUILD/"

# 编译（4MB flash / 1MB SPIFFS）
arduino-cli compile --fqbn "esp8266:esp8266:generic:FlashSize=4M1M" --output-dir /tmp/OmnixieBin "$BUILD"

# 产物 /tmp/OmnixieBin/Omnixiev1.1.5.ino.bin -> 命名为 webflash/firmware-vX.Y.bin (版本化)
# 再更新 manifest.json 的 path、latest.json 的 version/url/sha256/size
# sha256: shasum -a 256 webflash/firmware-vX.Y.bin
```

注意：为在现代工具链下编译通过，构建时对源码做了两处等价改动（不改行为）：
- `.ino` 里 `SoftwareSerial swSer(2,14,false,64)` 改为 3 参构造（第 4 参"缓冲 64"是新 API 默认值）。
- `SSDPDevice.cpp` 里 `IP2STR(&ip)` 改为手动按字节取 IP（适配 lwIP 变化）。

若坚持用原始 4 参写法，则需安装 ESP8266 core 2.3.0 那代自带的旧版 SoftwareSerial。
