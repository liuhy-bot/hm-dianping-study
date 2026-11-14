# 01 nginx 反向代理与双实例集群

## 目标

让 `/api` 的请求由两台后端实例共同处理，为后续验证「集群模式下的一人多单」做准备。

## nginx 配置要点

`conf/nginx.conf` 里需要两处配合：

```nginx
upstream backend {
    server 127.0.0.1:8081 max_fails=5 fail_timeout=10s weight=1;
    server 127.0.0.1:8082 max_fails=5 fail_timeout=10s weight=1;
}

server {
    listen 8080;

    location / {
        root   html/hmdp;
        index  index.html index.htm;
    }

    location /api {
        rewrite /api(/.*) $1 break;
        proxy_pass http://backend;      # 关键：指向 upstream 名，不能写死单台
    }
}
```

## 踩过的坑

### 坑 1：upstream 定义了但没生效

`proxy_pass` 还写着 `http://127.0.0.1:8081`，这时 upstream 定义得再完整也只是一段死代码，
所有请求都只打到 8081，8082 全程空闲。

判断方法：看两个实例的控制台，如果只有一边有请求日志，就是没走负载均衡。

### 坑 2：双击 nginx.exe 打不开

`nginx.exe` 是控制台程序，双击后窗口一闪而过属于正常现象——它已经在后台运行，
本来就没有图形界面。判断是否启动成功应该访问 `http://localhost:8080`。

停止和重启必须先结束进程：

```powershell
Stop-Process -Name nginx -Force
cd "E:\Reids learning\nginx-1.18.0"
.\nginx.exe
```

改配置后先校验再启动：

```powershell
.\nginx.exe -t
```

### 坑 3：error.log 里的报错不必都管

```
open() ".../.well-known/appspecific/com.chrome.devtools.json" failed
open() ".../js/axios.min.map" failed
```

第一条是 Chrome DevTools 的自动探测请求，第二条是 Source Map 文件缺失，
都属于无害噪音，不影响功能。

## 启动两个后端实例

在 IDEA 里复制一份运行配置，第二份加 VM 参数：

```
-Dserver.port=8082
```

操作路径：Run/Debug Configurations → 选中配置 → **Modify options (Alt+M)** →
勾选 Java 分组下的 **Add VM options (Alt+V)** → 在输入框里填上面那行。

IDEA 右上角的绿色三角只运行「下拉框当前选中」的那一个配置，
要跑两个必须切换下拉框分别启动，不能用同一个配置连点两次。

## 如何验证

三个端口能连通只是第一步，真正要确认的是**请求在两个实例间分发**：

1. 两个实例控制台分别打印 `Tomcat started on port(s): 8081 (http)` 与 `8082`
2. 连续访问几次 `/api/...` 接口，观察两个控制台的日志是否交替出现
3. 用 nginx 的 access 日志对照

## 附：本机端口检测工具不可靠记录

`Get-NetTCPConnection` 多次报告「8080/8081/8082 未监听」，而同一时刻接口请求全部正常返回 200。
改用 TCP 直连判断才得到可信结果：

```powershell
foreach ($p in @(8080,8081,8082)) {
    $c = New-Object System.Net.Sockets.TcpClient
    $iar = $c.BeginConnect("127.0.0.1", $p, $null, $null)
    $ok = $iar.AsyncWaitHandle.WaitOne(2000)
    if ($ok -and $c.Connected) { "PORT $p 通" } else { "PORT $p 不通" }
    $c.Close()
}
```
