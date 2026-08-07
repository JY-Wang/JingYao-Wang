# 黄金/白银实时行情浮窗

桌面常驻置顶小工具，数据来自「融通金」(jzj9999.com) 网页版的私有 WebSocket 行情网关。

## 目录结构

```
gold-silver-widget/
├── protocol.py     # 手写 protobuf 编解码（不依赖 protoc/grpcio-tools）
├── ws_client.py     # WebSocket 客户端：鉴权、订阅、心跳、断线重连
├── widget.py        # Tkinter 桌面浮窗界面
├── run.bat           # 双击启动（用 pythonw，不弹黑框）
└── requirements.txt
```

## 运行

```bash
pip install -r requirements.txt
python widget.py
```

或双击 `run.bat`。窗口可拖动，右上角 📌 切换置顶，✕ 关闭。

## 数据从哪来

页面 `https://i.jzj9999.com/quoteh5/` 是纯前端 SPA，价格不走普通 REST 接口，而是：

- **地址**：`wss://rtjwbqt.ytj9999.com:8443/gateway`
- **协议**：WebSocket 二进制帧 + protobuf（包名 `jadegold.msg.quotation.pbv2`）
- **鉴权**：连接后第一帧发送 `msgid=32 (auth)`，请求体里的 `token` 字段是

  ```
  Blowfish-CBC-PKCS5(
      key = "tdc5%y4yaU@xFi",
      iv  = "5X4f$^hp",
      明文 = "plaintract" + "rtj" + 当前毫秒时间戳
  )
  ```

  这些 key/iv/apptype/verifycode 都是**硬编码在网页自己的 JS 里**的，任何打开该页面的人都能拿到，不涉及破解权限。
- **订阅**：鉴权成功后发送 `msgid=18 (latestQuotation)`，请求体 `codes` 填商品代码（本项目订阅了 `JZJ_au_PS`/`JZJ_au_PB`/`JZJ_ag_PS`/`JZJ_ag_PB`，对应黄金/白银的销售价、回购价），`freq=[0]` 表示 REALTIME。
- **心跳**：之后每 20 秒发一个 `msgid=16 (heart_beat)` 空包，服务端超时不收心跳会断开连接。
- **推送**：服务端主动推送 `QuotationMsg`，其中 `response[].quotation[]` 是各订阅代码的最新行情，关键字段是 `rt.last`（最新价）、`rt.updown`（涨跌额）。

完整字段编号见 `protocol.py` 顶部的注释和各 `encode_*`/`decode_*` 函数，是照着对方 JS 里生成的 `encode()` 函数逐字段读出来的，不是猜的。

## ⚠️ 这套接口没有官方文档，随时可能失效

它是从网页前端代码里逆向出来的私有协议，不是公开 API。如果哪天浮窗一直显示"连接异常/重试中"，大概率是对方改了协议、换了域名，或者调整了鉴权方式。排查思路：

1. 打开 `https://i.jzj9999.com/quoteh5/?ivk_sa=1025883i`，浏览器里按 F12 打开开发者工具，看 `网络` 面板里静态资源文件名是不是变了（本项目写的时候是 `quoteh5-72b78336.js`、`DINfont-e08d4a7b.js`，文件名带 hash，改版后会变）。
2. 在浏览器控制台里执行下面代码，找当前的 WebSocket 地址（`DINfont-*.js` 文件里搜 `wss://`）：

   ```js
   fetch('/web/static/js/<当前文件名>.js').then(r=>r.text()).then(t=>{
     console.log([...new Set(t.match(/wss?:\/\/[^"'\s]+/g))]);
   });
   ```
3. 同一个文件里搜 `AUTH_DATA`，能直接看到 `apptype`/`verifycode`/`key`/`initializationVector` 四个字段的最新值。
4. 如果 protobuf 字段编号变了，可以在控制台里用下面方法把内部函数的源码直接摘出来看（比手动读混淆代码快得多）：

   ```js
   const mod = await import('/web/static/js/<DINfont文件名>.js');
   // 该模块把 protobuf 消息类挂在内部变量 $root 上，不是直接 export 的，
   // 需要先把源码文本里 export{ 之前插一行 const __ROOT=$root; 再包成 Blob URL 重新 import，
   // 具体做法见本项目开发过程中的做法（把相对 import 路径替换成绝对 URL 后再建 Blob）。
   ```

   拿到 `$root.jadegold.msg.quotation.pbv2.QuotationMsg.encode.toString()` 这类函数源码，
   里面每个 `t.uint32(N)` 的 N 就是 `(字段号<<3 | wire_type)`，照着改 `protocol.py` 里对应的
   `w_*`/`decode_*` 函数即可。
5. 也可以直接在控制台里跑：

   ```js
   const mod = await import('/web/static/js/<DINfont文件名>.js');
   mod.q({namespace:'test', codes:['JZJ_au_PS','JZJ_au_PB','JZJ_ag_PS','JZJ_ag_PB'],
          callback: console.log});
   ```

   如果控制台能正常打印出行情数据，说明订阅代码本身没变，问题只出在 Python 端的协议实现上；
   如果控制台也拿不到数据，说明网站那边改了东西，需要重新走一遍上面 1-4 步。

## 依赖

- `pycryptodome`（Blowfish 加密）
- `websocket-client`（同步 WebSocket）
- Python 3.10+ 自带 `tkinter`

## 已知限制

- 只订阅了黄金/白银的销售价与回购价（对应网页"黄 金"/"白 银"两行），没有接入黄金9999、T+D、伦敦金银等其它品种，如果需要可以在 `ws_client.py` 的 `CODES` 里加代码（代码名可以从网页 JS 的 `Ke` 对象里找，或抓包确认）。
- 没有做托盘图标/开机自启，需要的话可以后续加 `pystray`。
