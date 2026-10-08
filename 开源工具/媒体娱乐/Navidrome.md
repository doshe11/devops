以下是针对3M带宽ECS环境，使用Docker搭建Navidrome的完整教程。整个过程围绕**转码省流**这个核心目标展开。

---

## 一、环境准备

在开始之前，请确认你的ECS已满足以下条件：

- 已安装 **Docker** 和 **Docker Compose**
- 已将当前用户加入Docker组（可选）：`sudo usermod -aG docker $USER`，然后执行 `newgrp docker`
- 有一个存放音乐文件的目录（可以是ECS本地目录，也可以挂载NAS或对象存储）

---

## 二、创建目录结构

Navidrome需要两个目录：一个存放数据库和缓存（`/data`），一个存放音乐文件（`/music`）。

```bash
mkdir -p ~/navidrome/{data,music}
cd ~/navidrome
```

- `~/navidrome/data`：存放Navidrome的数据库、缓存和配置
- `~/navidrome/music`：存放你的音乐文件（如果你已有音乐库在其他路径，可以跳过，后续在compose中挂载对应路径即可）

---

## 三、编写 docker-compose.yml

在 `~/navidrome` 目录下创建 `docker-compose.yml`：

```bash
nano docker-compose.yml
```

填入以下内容：

```yaml
services:
  navidrome:
    image: deluan/navidrome:latest
    container_name: navidrome
    ports:
      - "4533:4533"
    restart: unless-stopped
    environment:
      ND_LOGLEVEL: info
      ND_SCANSCHEDULE: "24h"
      ND_SESSIONTIMEOUT: "6h"
      ND_BASEURL: "/navidrome"
      ND_ENABLETRANSCODINGCONFIG: "true"
      ND_TRANSCODINGCACHESIZE: "100MiB"
    volumes:
      - ./data:/data
      - ./music:/music:ro
```

### 关键配置说明

| 配置项                          | 作用                                            |
| :--------------------------- | :-------------------------------------------- |
| `user: "1000:1000"`          | 容器以指定用户运行，必须拥有 `data` 目录的读写权限和 `music` 目录的读权限 |
| `ND_ENABLETRANSCODINGCONFIG` | **核心配置**，开启转码功能，这是3M带宽环境下省流的关键                |
| `ND_TRANSCODINGCACHESIZE`    | 设置转码缓存大小，避免频繁重复转码，3M带宽下100MiB足够               |
| `ND_SCANSCHEDULE`            | 每24小时扫描一次音乐库，检测新增文件                           |
| `:ro`                        | 音乐目录以只读方式挂载，Navidrome不会修改你的原始文件               |

---


**常见错误排查**：

- **data目录不可写**：容器启动时报 `unable to open database file`，说明data目录权限不足
- **music目录不可读**：Web界面能打开但音乐库为空，日志出现 `permission denied`，说明容器用户无法读取音乐文件

> ⚠️ 不要使用 `PUID`/`PGID` 环境变量，Navidrome官方镜像**不支持**这些变量，必须使用 `user` 指令指定用户。

---

## 五、启动Navidrome

```bash
docker compose up -d
```

查看启动日志，确认没有报错：

```bash
docker compose logs -f
```

看到类似 `Starting Navidrome` 和 `Listening on 0.0.0.0:4533` 即表示启动成功。

---

## 六、初始访问与管理员设置

浏览器访问 `http://你的ECS公网IP:4533`，首次访问会提示创建管理员账号。设置好用户名和密码后进入Navidrome管理界面。

进入后，Navidrome会自动开始扫描 `/music` 目录中的音乐文件。首次扫描时间取决于音乐库大小，稍等片刻即可在界面中看到曲目。

---

## 七、转码配置（3M带宽核心优化）

这是本教程最关键的部分。Navidrome的转码配置分为**服务端**和**客户端**两个层面。

### 7.1 服务端：确认转码已启用

在 `docker-compose.yml` 中我们已经设置了 `ND_ENABLETRANSCODINGCONFIG: "true"`，这是开启转码的前提。启动后可以在Navidrome Web界面的 **设置 → 转码** 页面中确认转码预设是否存在。

### 7.2 服务端：设置默认转码码率

在Web界面中，进入 **设置 → 转码**，你会看到系统内置的转码预设。建议保留默认的MP3转码预设，并在用户设置中限制最大码率。

进入 **设置 → 用户 → 选择你的用户 → 个人设置**，找到 **最大比特率** 选项，将其设置为 **128kbps** 或 **192kbps**。这意味着当客户端请求高码率音频时，服务器会自动将其转码为较低码率的MP3流。

> 对于3M带宽（理论峰值约375KB/s），建议将最大比特率设为 **128kbps**，这样单用户流媒体仅占用约16KB/s带宽，留有充足余量供多用户或元数据请求使用。

### 7.3 客户端：移动网络下的码率限制

在手机端App（如音流、Symfonium、DSub等）中，需要单独设置网络环境下的音质策略：

- **移动网络下播放音质**：选择 **极高（320K）** 或 **标准（192K）**
- **Wi-Fi网络下播放音质**：选择 **无损（不转码）**

这样设置后，在移动网络下Navidrome会自动将FLAC等无损文件实时转码为MP3播放，大幅节省流量；回到Wi-Fi环境则直接播放原始无损文件，不影响音质。

### 7.4 验证转码是否生效

播放歌曲时，查看Navidrome容器日志：

```bash
docker compose logs -f navidrome | grep transcod
```

如果看到类似 `Transcoding` 的日志条目，说明转码正在工作。

---

## 八、可选：FFmpeg路径配置

Navidrome官方Docker镜像**已内置FFmpeg**，通常无需额外配置。但如果你在某些NAS或自定义环境中遇到转码失败，可以手动指定FFmpeg路径。

在 `~/navidrome/data` 目录下创建 `navidrome.toml` 文件：

```bash
nano ~/navidrome/data/navidrome.toml
```

添加以下内容：

```toml
FFmpegPath = "/usr/bin/ffmpeg"
```

然后重启容器：

```bash
docker compose restart
```

Docker版本的Navidrome会自动读取 `/data` 目录下的 `navidrome.toml` 配置文件。

---

## 九、常用管理命令

```bash
# 查看运行状态
docker compose ps

# 查看实时日志
docker compose logs -f

# 重启服务
docker compose restart

# 停止服务
docker compose down

# 更新Navidrome镜像并重建
docker compose pull && docker compose up -d

# 手动触发音乐库扫描（需在Web界面操作，或等待定时扫描）
```

---

## 十、3M带宽下的额外优化建议

1. **限制封面图片尺寸**：Navidrome可以配置封面图片的最大编码尺寸，默认已降至300px，减少图片传输带宽占用。如带宽紧张，可在 `navidrome.toml` 中添加 `CoverArtQuality = 75` 进一步压缩。

2. **关闭非必要的Gravatar头像**：在 `docker-compose.yml` 中设置 `ND_ENABLEGRAVATAR: "false"`，减少外部请求。

3. **使用OPUS格式转码（可选）** ：OPUS比MP3在同等音质下码率更低。如果你使用的客户端支持OPUS，可以在转码预设中将输出格式改为OPUS，进一步节省带宽。

4. **避免同时多用户高码率播放**：3M带宽下，建议同一时间只有1-2个用户以128kbps码率流媒体播放，否则可能占满带宽影响体验。

5. **考虑BBR加速**：在ECS上启用BBR拥塞控制算法，可以改善高延迟网络下的传输效率，减少卡顿。

---

完成以上步骤后，你就拥有了一个在3M带宽下高效运行的私有音乐流媒体服务器。核心要点是：**开启转码 + 设置最大比特率限制 + 客户端分网络环境配置音质**。这套组合可以让你的无损音乐库在移动网络下流畅播放，同时将带宽消耗控制在可接受范围内。