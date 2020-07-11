# Hits 计数服务

支持计数功能包括:

- 网页点击计数
- 网页会话计数
- 网页用户计数
- 网页点赞计数
- 网页五星评价

博客地址, [阅读 BLOG](https://gitdig.com/post/hits-counter/)

## 快速开始

````bash
$: docker run liujianping/hits:latest -h
Usage:
  main [command]

Available Commands:
  help        Help about any command
  server      server command

Flags:
  -a, --address string    http server listen address (default ":8080")
  -e, --email string      autocert register email address
  -h, --help              help for main
  -H, --hostname string   product instance (FQDN) hostname
  -o, --origins strings   origins allowed
      --prod              running on production mode
  -v, --verbose int       glog verbose
      --version           version for main
  -w, --work-dir string   service working directory (default ".")

Use "main [command] --help" for more information about a command.
````

服务默认情况下运行在开发模式下，切换到生产模式，增加<code>--prod</code>参数， 生产模式下，服务默认启动 `80/443` 两标准端口进行监听。标准端口必须对外开放，否则无法处理证书自动化签发。

参数说明：

- <code>-w</code> 参数， 设置服务工作路径，服务启动后，会在该目录下创建 `hits.db` 文件，存放 SQLite 数据。在生产模式下，还会创建 `certs` 子目录，用于 `https` 的证书自动化。
- <code>-H</code> 参数， 用于生产模式，设置主机访问域名，同时会自动化处理该域名主机的证书签发。
- <code>-o</code> 参数， 跨域请求支所支持的来源域名，如: -o=gitdig.com,www.gitdig.com 
- <code>-v</code> 参数， 服务运行日志级别

- **开发模式**

````bash
$: docker run -d liujianping/hits:latest server http -v 5
````

访问路径: http://localhost:8080

- **生产模式**

````bash
$: docker run -d  --network host -v ./hits:/app/hits liujianping/hits:latest server http  --prod -H api.gitdig.com -o www.gitdig.com -w /app/hits
````

生产模式下必须设置<code>-o</code> 参数，否则服务无法处理跨域请求。

## HTTP 接口说明

Protobuf 定义接口，再通过生成工具转化成 HTTP 接口。

````proto
syntax = "proto3";

import "options/options.proto";

package hits;

service Counter {
    option (options.service) = {
        version: "v1"
    };

    //Hit       触发点击计数、会话计数、用户计数
    rpc Hit(HitReq) returns (HitResp);
    //HitGet    只读点击计数、会话计数、用户计数
    rpc HitGet(HitGetReq) returns (HitGetResp);

    //ThumbUp   触发点赞计数
    rpc ThumbUp(ThumbUpReq) returns (ThumbUpResp);
    //ThumbGet  只读点赞计数
    rpc ThumbGet(ThumbGetReq) returns (ThumbGetResp);

    //Rate      五星评价计数
    rpc Rate(RateReq) returns (RateResp);
    //RateGet   只读五星评价
    rpc RateGet(RateGetReq) returns (RateGetResp);
}

message HitReq {
    string session_storage = 1; //会话存储 代表 Session 标记
    string persist_storage = 2; //持久存储 代表 User 标记
    string url = 3;
    int64 init_hits = 4;
    int64 init_ssns = 5;
    int64 init_uids = 6;
}

message HitResp {
    int64 hits = 1;
    int64 ssns = 2;
    int64 uids = 3;
    string session_storage = 4;
    string persist_storage = 5;
}

...
````

鉴于篇幅关系，只贴一个消息定义用于说明，完整定义，请参考项目仓库. 转化成为 HTTP API  接口后，就是这样：

- **HTTP 请求**

````bash
uri: /v1/hits.Counter/Hit 
method: POST
Origin: "YourOrigin"
Referer: "YourURL"
Content-Type: application/json; charset=utf-8
data:
    {
        url: "",
        sessionStorage: "",
        persistStorage: "",
        initHits: 0,
        initSsns: 0,
        initUids: 0,
    }
````

在请求头的部分增加了 **Origin** 与 **Referer** 头，是因为这两个头的信息对于网页计数来说比较关键，Origin 会用在跨域请求的验证阶段，Referer 则可以简化请求体中的 URL。如果 URL 字段没有填写的话就可以使用  Referer 字段代替。当然前提是 Referer Police 的策略设置正确。

- **HTTP 响应**

````bash
http status: 200
http response data:
{
    "hits":"9",
    "ssns":"13",
    "uids":"1",
    "sessionStorage":"MTU5NDQzMDk5NnxoY1RhSFZBZ1NSMWFhbkpFUUZPcUV3VUtKOFBJWVJHdVVwcXBHWFY1d0UzZmpaU0Q3Njg2TmVLTmw0c2lZaE5JZWV3WXlJdFZhUFdEbzZUR2xKTFpxNUQ4TjR1SDVNNkhtSGVKWlY1MTBOS3ZlZHJCQjZSX1M2cklSbm4yNGVHZ09TRT18pOwHfdrgcI6zbhsMgKP7StfhOHVBfSa9yNY98cboZP4=","persistStorage":"MTU5NDM3NzEzMXxna3JOU1BhTk5FWVZfQm1CU01RaDNLb0pIdGhfQnZyV0F6THdVbHdrM0FjUGFrSlRoakVWeE1DdUF6T0dxM2QxY1VZZ01GSTl1dGhVa0tpVW51OXhRR3g0dU1ZODFDUVdLUnpueXdkU25hOENkX0k1TzJ4Mm9TNVNnVzZqR1hKOW43QT18yvUZPmys-LSUEt1H-UD6Weox0dkpDBSZ7clYDw0Rt0A="
}
````

## 常见问题

> **问：**Referer 头在请求头中没有出现？

referer 头没有出现的原因，是当前网页设置的 referrer-policy 决定的。为保证referer可以传递网页地址，请在网页元数据中添加以下代码:

````html
<meta name="referrer" content="no-referrer-when-downgrade" />   
````

> **问：**计数数据如何导出？

服务数据存储采用的是SQLite，存储路径在服务的工作目录下的`hits.db`文件。你可以直接取得该文件，利用 SQLite 官方提供的命令行工具进行访问导出。

````bash
$: sqlite3 hits.db .dump > hits.sql
````

> **问：**如何验证服务正常？

在生产模式下，服务提供一个状态检查的访问地址 <code>https//your.domain/ping</code>, 在浏览器中访问改地址，正常情况下会返回`pong`的响应，说明一切正常。

> **问：**为什么生产模式下容器的网络模式是主机(host)共享模式

计数服务会统计来源机器的真实IP地址，在非主机(host)共享的网络模式下，服务拿到的 IP 地址是容器的虚拟地址，所以必须将容器的网络模式设置为主机(host)共享模式。

> **问：**没有云主机怎么办？

没有云主机的话，目前暂时无法使用该服务。你可以在该页面下留言，或是发邮件联系我，我可以将你的域名加到我提供的服务中。如果很多人需要的话，我会考虑将该服务做成一个免费的服务，不过需要朋友们的赞助支持。

## 开源说明

为了拉点人气，目前项目以镜像的方式发布并提供接口目录。点赞数 > 100 时，完全开源。所以，支持一下吧。