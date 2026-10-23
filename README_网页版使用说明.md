# ADT 网页版 · 使用说明

复现对象:**Galla et al. (2014)** 的 Academic Diligence Task(ADT)。
纯原生 HTML/JS,零依赖,被试打开网页即可测试,无需安装任何软件。

## 一、怎么跑起来

**方式 A(本地双击)**:直接双击 `index.html`,浏览器打开即可。
   - 视频在 `videos/` 文件夹里,与 `index.html` 同目录即可正常播放。

**方式 B(部署给被试,推荐)**:把整个 `ADT_web` 文件夹上传到任意静态托管
   (GitHub Pages、学校服务器、Nginx 等),把网址发给被试即可。
   - 本机快速起一个服务器测试:`python -m http.server 8000`(在 `ADT_web` 文件夹下运行),
     然后浏览器打开 `http://localhost:8000`。

> 建议用方式 B:通过 `http://` 访问时,视频和自动提交最稳定;`file://` 直接打开一般也能跑,
> 但个别浏览器对本地视频/跨域提交有限制。

## 二、改配置(都在 `index.html` 顶部的 `CONFIG` 里)

| 项 | 说明 |
|---|---|
| `practiceSeconds` / `blockSeconds` / `nBlocks` | 练习时长 / 每块时长 / 块数(默认 60 / 240 / 5) |
| `allowNegative` | 数学题是否出现负数(默认 false) |
| `submitUrl` | **你的数据接收地址**,如 `https://你的服务器/api/adt` |
| `submitMethod` | 默认 `POST` |
| `submitFormat` | `"json"`(提交 JSON 给后端)或 `"form"`(表单编码,适配 Google Forms / Apps Script) |
| `videos` | 视频文件路径列表,有几个就有几个下拉选项(以文件名命名,带声音) |

## 三、数据长什么样

提交的数据是一个 JSON,结构如下:

```json
{
  "meta": { "participant": "001", "session": "001", "submittedAt": "...", "ua": "...", "screen": [1920, 1080] },
  "blocks": [ { "block": 1, "time_math": 120.5, "time_tetris": 60.1, "time_video": 59.4, "n_attempted": 30, "n_correct": 29 } ],
  "problems": [ { "block": 1, "problem": "7-3", "response": "4", "correct": 1 } ],
  "switches": [ { "t": 12.3, "from": "math", "to": "tetris" } ]
}
```

- **核心因变量**:`blocks` 里各块 `n_correct` 之和 = 生产力;`time_math` 占比 = 时间投入。
- 结束页还会有一个「下载数据(CSV)」按钮作为兜底。

## 四、如何接收数据(服务端示例)

### 示例 1:Python(Flask)后端

```python
from flask import Flask, request
app = Flask(__name__)

@app.route("/api/adt", methods=["POST"])
def adt():
    data = request.get_json(force=True)   # 对应 submitFormat="json"
    # 存到文件
    import json, os
    os.makedirs("data", exist_ok=True)
    fn = f"data/ADT_{data['meta']['participant']}_{data['meta']['submittedAt']}.json"
    with open(fn, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)
    return "ok", 200
```

### 示例 2:Google Apps Script(用"表单"方式)

1. 新建 Google 表格 → 扩展程序 → Apps Script,粘贴:

```js
function doPost(e) {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("Sheet1");
  const p = e.parameter;                 // 对应 submitFormat="form"
  sheet.appendRow([new Date(), p.participant, p.session, p.data]);
  return ContentService.createTextOutput("ok");
}
```

2. 部署为「Web 应用」→ 允许「任何人」访问 → 得到网址填到 `submitUrl`,
   并把 `submitFormat` 改成 `"form"`。

> 注意:提交请求跨域时,你的服务端要允许 CORS(Flask 可加 `flask-cors`,
> 或返回头 `Access-Control-Allow-Origin: *`)。若用 Google Apps Script 则无此问题。

## 五、常见问题

1. **视频没声音** → 浏览器默认可能拦截自动播放;被试点过「开始实验」和下拉菜单后即可有声播放。
   若仍无声,把视频转成 H.264 + AAC 的 mp4。
2. **视频加载失败** → 检查 `videos/` 里文件名是否与 `CONFIG.videos` 里的路径完全一致(含中文和扩展名)。
3. **数据提交失败** → 看结束页提示;确认 `submitUrl` 正确、服务端允许跨域;可用「下载 CSV」兜底。
4. **移动端** → 数学输入需要实体键盘,建议用电脑浏览器施测。
