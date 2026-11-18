# tsmap 使用笔记（DB 固晶工程师版）

> 官方仓库：https://github.com/wafertools/tsmap
> 浏览器在线版：https://wafertools.github.io/tsmap/
> 配套引擎：`@wafertools/wafermap`（npm，可嵌入自建系统）
> 许可：MIT 免费开源（作者 Paul Robins）

## 一、它是什么

tsmap 是一个**晶圆图（wafer map）查看与分析工具**：

- **桌面版**：Tauri v2 构建，Windows / macOS / Linux 都能装，不用联网。
- **网页版**：浏览器直接打开，无需安装（解析器编译成 WASM 在本地跑）。
- **输入**：STDF、ATDF（探针台/测试机标准数据），以及 CSV / JSON / Parquet；支持 .gz / .bz2 / .zip 压缩包。
- **输出**：晶圆图（可导出 PNG）、良率与 Bin 分布、空间规律分析、各类图表、数据表格（可导出 CSV）。

> ⚠️ 一句话定位：它是**看“结果分布”**的工具，不是固晶机软件，也读不了固晶机日志。

## 二、和 DB（装片）工作怎么结合

### 用法 1：固晶前看 CP/FT 良率，决定跳 die
- 用 tsmap 打开 CP 测试 STDF，先看**哪片晶圆良率低、不良 die 集中在哪个区域**（边缘环状、中心簇、某象限、条状）。
- 把这些区域和固晶机的 **skip die / map 功能** 对上，只对好 die 点胶固晶：
  - 省银胶 / DAF 膜
  - 减少固化后才发现的不良
  - 提高有效 UPH

### 用法 2：把 DB 不良“画”到晶圆上看规律（最实用）
DB 工序本身不做电测，但你可以把**自己检出的不良**整理成 CSV，当成 wafer map 打开：

| 不良项 | 建议字段 | 能看出什么 |
|---|---|---|
| 推力不足 / Die Shear 偏低 | 推力值 gf、x、y | 是否集中在晶圆某侧（顶针高度、吸嘴贴合、加热台温差） |
| 空洞 / 气泡（DAF、银胶） | 空洞面积 %、x、y | 与晶圆位置、边缘翘曲、点胶路径的相关性 |
| 偏位 / 旋转 | 偏移量 um、角度、x、y | 是固定方向偏（视觉标定）还是随机偏（送料/吸附） |
| 芯片崩边 / 裂纹 | 不良代码、x、y | 是否集中在晶圆 edge die（减薄/切割残留应力） |
| 溢胶 / 爬胶 | 溢胶长度、x、y | 与点胶量、加热台温度分布的关系 |

看的是**空间聚类**：
- 边缘集中 → 晶圆翘曲、顶针高度、吸附偏心、边缘 die 厚度差异
- 中心集中 → 加热台中心温差、点胶针头高度
- 条状 / 同一步进场重复 → reticle 或 wafer 前道相关问题（tsmap 的 **Reticle…** 功能专门找“每个场都重复失效”的位置）
- 全片随机 → 大概率是机台稳定性、吸嘴污染、银胶批次

### 用法 3：换料 / 换参数前后对比
用 **Wafer splits** 功能给每片晶圆打上分组标签（DAF膜A版/B版、银胶型号、顶针高度、固晶压力组），再把所有图表按分组对比，改善前后一眼看出差异。

## 三、支持的格式

| 格式 | 解析 | 说明 |
|---|---|---|
| STDF | Rust 内置解析 | 二进制 V4：MIR（批次信息）、WIR/WRR（晶圆）、PIR/PRR（die 的 hbin/sbin/x/y）、PTR/FTR（测试值） |
| ATDF | Rust 内置解析 | ASCII V4，结构同 STDF |
| CSV | Rust | 打开前有**列映射**步骤，支持宽表与长表（pivot） |
| JSON | Rust | 平铺数组或嵌套 `[{wafer字段, results:[{die}]}]` |
| Parquet | Rust | 列式，同样走列映射 |

## 四、四种打开方式

1. **网页版（最快）**：直接开 https://wafertools.github.io/tsmap/ ，把文件拖进去。
   > ⚠️ 公司真实晶圆数据、客户料号不建议往公网网页版传，涉及保密。
2. **桌面版（推荐厂内用）**：
   - Windows：装 `tsmap-<版本>-windows-x64.msi`。安装包**没有代码签名**，会弹“Windows 保护了你的电脑” → 点「更多信息」→「仍要运行」。
   - macOS：装好后在 Finder 里**右键 → 打开**（只需一次）；若提示“已损坏”，终端执行 `xattr -dr com.apple.quarantine /Applications/tsmap.app`。
   - Linux：AppImage 先 `chmod +x` 再运行；也有 .deb / .rpm。
3. **自建内网部署（最合规）**：每个 release 附带 `tsmap-<版本>-web.zip` 静态包，直接丢到公司内网服务器，全厂浏览器访问，数据不出厂。
4. **命令行 / URL 打开**：
   - 桌面：`tsmap --url <数据地址> --url-format <格式>`，需要鉴权头时加 `--url-headers <文件>`
   - 浏览器：`?dataUrl=...&dataFormat=...`
   - 自定义深链：`tsmap://open?url=...`

## 五、CSV 给 DB 用的最小字段（自己造数据时照这个来）

```
wafer_id,x,y,bin,thrust_gf,void_pct
LOT01-W01,12,35,PASS,620,1.2
LOT01-W01,13,35,FAIL_THRUST,380,2.5
```

- `wafer_id`：晶圆号（同时放批次号更清楚）
- `x` / `y`：die 坐标或行列号。**没有坐标只能出 die 列表，画不出晶圆图**
- `bin`：结果（合格 / 不良代码）。文字代码也能用，但要先在 Bin 定义里映射 PASS/FAIL
- 数值列：推力、空洞面积、偏移量等，可直接做 bin pareto 之外的箱线图 / 相关性矩阵

## 六、必做的四处设置（不做结果会错）

1. **Setup ▾ → 晶圆直径 + 边缘排除带（edge exclusion, mm）**
   边缘 die 算不算，直接决定良率口径，要和厂内 CP 报表一致。
2. **Setup ▾ → Bin 定义（Bin definitions…）**
   指定哪些 bin 是 PASS、并给硬 bin / 软 bin 起名，否则良率算不准。
3. **重测策略（retest policy）**
   同一 die 测过多次，默认取**最后一次**结果；若厂内口径是首次或最优，记得改。
4. **有效量程（Validity limits）**
   把测试机 clamp 值、开路读数（例如 9999）排除掉，单独计数，别混进统计。

## 七、主要看板功能

- **Summary / Stats**：良率、Bin 分布、环状与象限分析、空间 findings
- **Charts & Insights**：yield by wafer、bin pareto、单测试箱线图与直方图、交叉测试相关性矩阵
- **Plot 标签页**：自建散点 / 直方 / 箱线 / 柱状 / 折线 / 响应曲线，可对任意测试值、die 字段、晶圆属性作图；图表可导出导入，换电脑也能带走
- **Data 标签页**：整批 die 表格、每片晶圆一行汇总，可 CSV 导出；框选一批 die 只对它们作图或列表
- **Reticle…**：画步进场网格，找每个场都重复失效的位置
- **Scan a folder**：指向一个几百个文件的目录，只扫表头元数据（批号、产品、机台、片数、日期）成可筛选表格，再挑几片真正要看的载入
- **PNG 导出**：任意晶圆图直接存图，贴 8D、改善报告、PPT 都好用

## 八、DB 现场三条实操流程

### 流程 A：看 CP 良率决定跳 die
1. 拿到 CP 的 STDF / ATDF（或让测试给 CSV）
2. 载入 → 设直径、边缘排除 → 确认 Bin 定义
3. 看 bin pareto + 每片良率，找出低良率片与不良聚集区
4. 导出晶圆图 PNG，附到改机/试产报告

### 流程 B：查一批 Die Shear 不良到底是哪个位置出问题
1. 把不良 die 的坐标、推力值整理成 CSV（见上面最小字段）
2. 载入 → 列映射（wafer id / x / y / 推力值）
3. 设直径与边缘排除 → 用 Plot 标签页做“推力值 vs x/y”的散点，或直接看 bin 分布图
4. 判断：边缘集中 → 查顶针高度、吸附偏心、加热台；中心集中 → 查点胶针高、中心温差；随机 → 查吸嘴污染、银胶批次
5. 导出图 + Data 表格 CSV，写进 8D 的 D4 原因分析

### 流程 C：DAF 膜 A/B 版对比
1. 两版各挑若干片，CSV 里 wafer_id 带上分组后缀（或载入后用 splits 打标签）
2. 按 splits 分组看良率、空洞率箱线图
3. 结论直接落到“换不换料”的决策

## 九、常见坑

- **没坐标画不出图**：只有 die 列表时别以为是软件坏了
- **CSV 列名不统一**：一定要走列映射，别硬编码列名
- **直径/边缘排除没设**：良率数字和厂内报表对不上，白忙一场
- **数据合规**：真实客户数据走桌面版或内网自建版，不要传公网网页版
- **别指望它读固晶机日志**：固晶机数据要先导出成 CSV 才能进来
- **它也不是 DOE 工具**：空间规律归 tsmap，参数因果归 Minitab/DOE

## 十、工具分工（建议）

| 需求 | 用什么 |
|---|---|
| 不良在晶圆上的空间规律 | **tsmap** |
| 参数与不良的因果关系（压力/温度/时间/点胶量） | Minitab DOE、回归、CPK |
| 固晶机内部 map 跳 die | 设备自带 map 功能（ASM / 新益昌 / 艾科瑞斯） |
| 报表与 8D 图片 | tsmap 导出 PNG + Excel |

---

## 参考链接
- tsmap 仓库：https://github.com/wafertools/tsmap
- 文档与在线版：https://wafertools.github.io/tsmap/
- wafermap 引擎（自建系统嵌入用）：npm `@wafertools/wafermap`
- 数据解析器：npm `@wafertools/testdata-parser`
