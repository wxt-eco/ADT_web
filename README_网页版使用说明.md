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

| 项                                                   | 说明                                                                                |
| ---------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `practiceSeconds` / `blockSeconds` / `nBlocks` | 练习时长 / 每块时长 / 块数(正式版 60 / 240 / 5;当前测试用 10 / 10 / 5)              |
| `allowNegative`                                    | 数学题是否出现负数(默认 false)                                                      |
| `submitUrl`                                        | **你的数据接收地址**,如 `https://你的服务器/api/adt`                        |
| `submitMethod`                                     | 默认 `POST`                                                                       |
| `submitFormat`                                     | `"json"`(提交 JSON 给后端)或 `"form"`(表单编码,适配 Google Forms / Apps Script) |
| `videos`                                           | 视频文件路径列表,有几个就有几个下拉选项(以文件名命名,带声音)                        |
| 内置小游戏                                           | 俄罗斯方块、扫雷已内置,无需配置,自动出现在下拉菜单里                                |

## 三、数据长什么样

提交的数据是一个 JSON,结构如下:

```json
{
  "meta": { "participant": "001", "session": "001", "submittedAt": "...", "ua": "...", "screen": [1920, 1080] },
  "blocks": [ { "block": 1, "time_math": 120.5, "time_tetris": 60.1, "time_video": 59.4, "time_minesweeper": 0, "n_attempted": 30, "n_correct": 29, "accuracy": 0.967 } ],
  "problems": [ { "block": 1, "problem": "7-3", "response": "4", "correct": 1 } ],
  "switches": [ { "block": 1, "t": 12.3, "from": "math", "to": "tetris" } ],
  "ratings": [ { "block": 1, "boredom": 2 } ]
}
```

- **核心因变量(与原文一致)**:时间投入 = 数学时间占比 `sum(time_math) / sum(time_total)`;生产力 = `sum(n_correct)`。
- `accuracy`(正确率)= `n_correct / n_attempted`;原版约 96%,区分度主要在时间分配而非正确率。
- `ratings` 是每个块结束后的无聊程度评定(`boredom` 为 1-4)。
- 结束页的「下载数据(CSV)」按钮会导出多段 CSV:表1 块级汇总(每块的数学/俄罗斯方块/视频/扫雷时间分配 + 答题数 + 正确率 + 无聊程度 + 一行合计)、表2 逐题明细、表3 切换记录。

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

### 示例 2:Google Apps Script + Google 表格(推荐,免费、无需服务器)

1. 新建一个 Google 表格(建议命名 `ADT数据`)。
2. 菜单 **扩展程序 → Apps Script**,清空默认内容,粘贴:

```js
function doGet() {
  return ContentService.createTextOutput('ADT 服务正常,数据会写入本表格');
}

function doPost(e) {
  const p = e.parameter || {};
  const participant = String(p.participant || '');
  const session = String(p.session || '');
  let data = {};
  try { data = JSON.parse(p.data || '{}'); } catch (err) { data = {}; }

  const ss = SpreadsheetApp.getActiveSpreadsheet();

  // 表1:拆分列(每块一行),便于直接统计
  const HEADER = [
    '实验开始时间', '实验结束时间', '被试编号', '场次', '第几块',
    '数学时长_秒', '俄罗斯方块_秒', '视频_秒', '扫雷_秒', '总时长_秒',
    '答题数', '正确数', '正确率', '无聊程度', '屏幕宽', '屏幕高'
  ];
  const sheet = getSheet(ss, '数据', HEADER);
  sheet.getRange('C:D').setNumberFormat('@');   // 被试编号/场次 → 纯文本,避免 001 变 1

  const meta = data.meta || {};
  const startLocal = meta.startedAtLocal || '';
  const endLocal = meta.submittedAtLocal || '';
  const screen = (meta.screen && meta.screen.length) ? meta.screen : ['', ''];
  const blocks = data.blocks || [];
  const ratings = data.ratings || [];

  const rows = [];
  if (blocks.length === 0) {
    rows.push([startLocal, endLocal, participant, session, '', '', '', '', '', '', '', '', '', '', screen[0], screen[1]]);
  } else {
    blocks.forEach(function (b) {
      const r = ratings.find(function (x) { return x.block === b.block; }) || {};
      const timeTotal = (b.time_math || 0) + (b.time_tetris || 0) + (b.time_video || 0) + (b.time_minesweeper || 0);
      rows.push([
        startLocal, endLocal, participant, session, b.block,
        b.time_math, b.time_tetris, b.time_video, b.time_minesweeper,
        timeTotal, b.n_attempted, b.n_correct, b.accuracy, r.boredom, screen[0], screen[1]
      ]);
    });
  }
  if (rows.length) {
    sheet.getRange(sheet.getLastRow() + 1, 1, rows.length, HEADER.length).setValues(rows);
  }

  // 表2:原始 JSON(完整备份,可还原任何字段)
  const raw = getSheet(ss, '原始JSON', ['提交时间', '被试编号', '场次', '完整JSON数据']);
  raw.getRange('B:C').setNumberFormat('@');
  raw.appendRow([new Date(), participant, session, p.data || '']);

  return ContentService.createTextOutput('ok');
}

function getSheet(ss, name, header) {
  let s = ss.getSheetByName(name);
  if (!s) s = ss.insertSheet(name);
  if (header && header.length) {
    // 每次运行都把第 1 行表头写成最新版,防止旧表头残留错位
    s.getRange(1, 1, 1, header.length).setValues([header]).setFontWeight('bold');
    s.setFrozenRows(1);
  }
  return s;
}
```

3. 点 **部署 → 新部署 → 类型选「Web 应用」**;「执行身份」= 我,「访问权限」= **任何人**,部署并授权。
4. 复制 **Web 应用网址**(形如 `https://script.google.com/macros/s/XXXX/exec`),填到 `submitUrl`,
   并确认 `submitFormat` 为 `"form"`(已在网页版里内置了 `no-cors` 发送,跨域可正常送达)。
5. 之后若改了脚本代码,需在「部署 → 管理部署 → 编辑 → 新版本」重新部署,网址不变。

> 提交后会自动生成两个标签页:
>
> - **数据**:拆分好的列,每块一行(实验开始/结束时间、被试编号、场次、第几块、各游戏/视频时长、总时长、答题数、正确数、正确率、无聊程度、屏幕尺寸),可直接做统计。
> - **原始JSON**:完整备份,一行一条提交,最后一列是该被试的完整 JSON,任何字段都能从这里还原。
>
> 首次使用前,建议把表格里旧的「数据」「原始JSON」两个标签页删掉,让脚本用新表头重新生成。

> 注意:若改用自建后端(Flask/Node 等),`submitFormat` 用 `"json"`,且服务端要允许 CORS
> (Flask 可加 `flask-cors`,或返回头 `Access-Control-Allow-Origin: *`)。

## 五、常见问题

1. **视频没声音** → 浏览器默认可能拦截自动播放;被试点过「开始实验」和下拉菜单后即可有声播放。
   若仍无声,把视频转成 H.264 + AAC 的 mp4。
2. **视频加载失败** → 检查 `videos/` 里文件名是否与 `CONFIG.videos` 里的路径完全一致(含中文和扩展名)。
3. **数据提交失败** → 看结束页提示;确认 `submitUrl` 正确、服务端允许跨域;可用「下载 CSV」兜底。
4. **移动端** → 数学输入需要实体键盘,建议用电脑浏览器施测。
5. **扫雷** → 已内置。左键开格、右键插旗、点顶部笑脸重开。
