[Immich](https://immich.app/) 是一个开源的照片/视频管理方案，它的功能和用途类似于 Google 相册或 iCloud 照片。任何个人都可以利用 Immich 搭建自己私有的云同步相册，并支持多端/多人使用。

## 部署 Immich

```bash
mkdir -p ./immich
cd ./immich
# 下载必要的文件 
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
```

修改环境变量 (.env)
```bash
IMMICH_VERSION=v3.0.1  # 确保版本锁定在 v3.0.1
UPLOAD_LOCATION=/path/to/your/photos # 修改为您实际存放照片的绝对路径
DB_DATABASE_NAME=immich
DB_USERNAME=immich
DB_PASSWORD=your_secure_password
DB_HOSTNAME=postgres
```
Immich v3 系列引入了全新的 **VectorChord** 数据库插件支持。请打开 `docker-compose.yml` 文件，检查并确保 PostgreSQL 数据库使用的镜像是 `ghcr.io/immich-app/postgres:16-vectorchord0.3.0-pgvectors0.3.0` 或最新版本。 

```bash
docker compose pull 
docker compose up -d
```

### 更新到最新版本

Immich 平均一周会发一个小版本, 查看更新说明: [immich/releases](https://github.com/immich-app/immich/releases)

更新到最新版本只需要执行如下命令即可

```bash
# 先进到应用的目录
cd ~/Portable/immich-app

docker compose pull && docker compose up -d
```

如果想要更新到指定的版本，修改 `.env` 内的 IMMICH_VERSION 为版本号后再升级，如: v1.100.0

## Immich 功能配置   

### 存储模板配置

这个功能默认是关闭的，根据官方文档中关于《[存储模板的实现方式](https://immich.app/docs/administration/backup-and-restore#asset-types-and-storage-locations)》的解释，开启后可以自定义图片存储结构。**适合文件夹强迫症用户。

以本文将图片挂载到 `/photos` 文件夹为例。不论你是否启用存储模板，`/photos` 文件夹中都会生成如下文件夹：

- `profile`：用户个人资料图片
- `thumbs`：存储缩略图
- `encoded-video` ：存储转码视频
- ==`upload`==：**存储上传的图片，启用存储模板后，这个文件夹会变成临时中转站**
- `library`：启用存储模板后，图片会从 `upload` 移动至此

假如没有启用存储模板，即默认设置的状态下，图片会保存在 `/photos/upload/<userID>` 文件夹中，`userID` 可以在账户设置 -> 账户 -> 用户 ID 中找到，这个 ID 为字符串且唯一。

如果启用了存储模板，所有图片会被自动任务移动到 `/photos/library/<userID>` 文件夹中。这个 `userID` 和上面的情况一样，都是唯一的字符串。但管理员可以为用户额外设置**存储标签**（管理员有一个默认的 `admin` 存储标签。），该标签被设置后将会代替 `<userID>`。也就是说，图片会被保存到 `/photos/library/admin` 文件夹。

接着选择自己喜欢的存储模板。我选择的是 `{{y}}/{{MM}}/{{dd}}/{{filename}}`，即最终图片会被保存在结构为 `/2022/02/03/IMAGE_56437.jpg` 的文件夹中。

### 配置地理信息（Geodata）

为解决 Immich 默认地理反编码结果为英文的问题，可以使用社区项目 [immich-geodata-cn](https://github.com/ZingLix/immich-geodata-cn) 实现中文地址显示。

**特点：**
- 国内地址全面汉化（含港澳台）
- 实验性支持海外地址（如日本）
- 地址规范化为标准四级行政区格式（通过高德/Nominatim API）

**挂载方式示例：**
```bash
/data/application/immich/server/geodata:/app/immich/server/geodata:rw    
/data/application/immich/server/i18n-iso-countries:/app/immich/server/node_modules/i18n-iso-countries:rw
```


## 批量从服务器本地导入文件

在 im­mich 设置中创建一个 api key 
![[开源工具/assets/efd15d63b0a2ef1a262d3fd83ffec2ec_MD5.webp|700]]

使用 `immich login <url> <key>` 的格式连接到 im­mich，例如：
```bash
docker exec -it immich /bin/bash

immich login http://127.0.0.1:2283 Cn5BmZWKQ5pQgCVaQjwIjDs9
Logging in to http://127.0.0.1:2283
Discovered API at http://127.0.0.1:2283/api
Logged in as admin@immich.com
Wrote auth info to /root/.config/immich/auth.yml

# 随后就可以直接上传图片了：
immich upload file1.jpg file2.jpg

# 也可以上传一整个文件夹。默认不包括子文件夹，如需上传包含子文件夹的文件夹，需加上 `--recursive`：
immich upload --recursive directory/

# 默认情况下，upload 命令将在上传文件之前对文件进行哈希处理。这是为了避免多次上传同一个文件。如果确定文件是唯一的，则可以通过传递该选项来跳过此步骤。请注意，Immich 始终通过哈希执行自己的重复数据删除，因此这只是一个性能考虑因素。如果您有良好的带宽，跳过哈希可能会更快。`--skip-hash`
immich upload --skip-hash --recursive directory/


# 使用 `--album-name` 上传到指定相册：
immich upload --album-name "初音未来" --recursive miku/

```