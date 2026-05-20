# BDD 节点概率 XML 输出修改说明（name-based，无割集）

## 需求更新
1. BDD 算法输出所有节点概率到 XML 报告，**节点标识使用基本事件的 name**，而不是 index
2. **不输出 `<sum-of-products>`（割集）**
3. 没有编译环境，如何验证/使用

## 修改文件

| 文件 | 修改内容 |
|------|----------|
| `src/probability_analysis.h` | 将 `bdd_node_probabilities()` 返回类型从 `pair<int, double>` 改为 `pair<string, double>` |
| `src/probability_analysis.cc` | `CollectNodeProbabilities` 中通过 `graph()->basic_events()[ite.index()]` 获取 `mef::BasicEvent*`，存入 `event->id()`（即 name）。Module（中间 gate）节点被跳过，只保留基本事件节点。 |
| `src/reporter.cc` | ① 清空 `ReportResults(FaultTreeAnalysis, ...)` 函数体，**彻底禁用 `<sum-of-products>` 输出**；② `ReportResults(ProbabilityAnalysis, ...)` 中 `<node>` 属性改为 `name="..."` |
| `share/report.rng` | `<node>` 的 `index` 属性改为 `name`（NCName 类型） |

## 预期 XML 输出示例

```xml
<results>
  <bdd-nodes name="TopEvent">
    <node name="B" probability="0.28"/>
    <node name="A" probability="0.1"/>
  </bdd-nodes>
</results>
```

- 不再包含 `<sum-of-products>`、`<product>`、`<literal>` 等割集相关元素
- `<bdd-nodes>` 直接挂在 `<results>` 下
- 每个 `<node>` 的 `name` 对应输入模型中 `<basic-event>` 的 `name` 属性

## 关于"替换编译后文件就能运行"

**很遗憾，C++ 不支持直接替换源码文件后运行**，必须经过编译和链接。但有以下几种不同程度的简化方案：

### 方案 A：只重编译修改的 3 个文件（需有编译器+链接器）

如果你身边有同事或服务器带有编译环境，不需要全量编译，**只需要重新编译被修改的 3 个源文件，然后重新链接**即可：

```bash
# Linux/MinGW 示例（假设已有 build 目录和 cmake 缓存）
cd build

# 仅重新编译修改的 3 个 .cc 文件
make -j$(nproc) \
    CMakeFiles/scram_lib.dir/src/probability_analysis.cc.o \
    CMakeFiles/scram_lib.dir/src/reporter.cc.o

# 重新链接 scram 可执行文件
make -j$(nproc) scram
```

> 实际上 `make` 会自动检测修改，你只需运行 `make -j$(nproc) scram` 即可，它会只编译变更文件。

### 方案 B：替换完整可执行文件（最实际）

如果你**完全没有任何编译工具**，唯一可行的方式是：
- 把修改后的源码发给有编译环境的人
- 对方编译完成后，把生成的 `scram.exe`（Windows）或 `scram`（Linux/macOS）完整可执行文件发给你
- **你直接用新可执行文件替换旧的即可运行**

这就是 C++ 的"替换编译后文件"——替换的是**完整的可执行文件**，而不是中间 .o 文件。

### 方案 C：使用 Docker（你只需有 Docker，无需 C++ 环境）

项目自带 `Dockerfile`，只要你有 Docker 就能编译出可执行文件：

```bash
# 1. 用 Dockerfile 编译（这一步在 Docker 容器内自动完成）
docker build -t scram-bdd-nodes .

# 2. 把编译好的二进制从镜像里拷出来
docker create --name extract scram-bdd-nodes
docker cp extract:/usr/local/bin/scram ./scram_new

# 3. 直接用 scram_new 替换你原来的 scram 可执行文件
./scram_new --bdd --probability -o report.xml your_model.xml
```

### 方案 D：GitHub Actions / CI 编译

把修改 push 到 GitHub，利用项目已有的 `.travis.yml` 或自建 GitHub Actions Workflow，CI 会自动编译。编译完成后，你可以从 CI 的 Artifacts 中下载可执行文件。

---

## 没有编译环境，如何验证逻辑正确性？

### 方法一：Python 脚本验证（已提供）

运行项目目录下的 `verify_bdd_nodes.py`：

```bash
python verify_bdd_nodes.py
```

它会模拟：
- BDD 概率递归计算（与 C++ `CalculateProbability` 逻辑等价）
- name-based 的节点收集（与 C++ `CollectNodeProbabilities` 逻辑等价）
- XML 格式化输出（与 C++ `Reporter` 逻辑等价）
- **验证不包含 `<sum-of-products>`**

输出示例：
```xml
<results>
  <bdd-nodes name="TopEvent">
    <node name="B" probability="0.28000000000000003" />
    <node name="A" probability="0.10000000000000001" />
  </bdd-nodes>
</results>
```

如果脚本输出 **"验证通过"**，说明修改逻辑在算法层面是正确的。

### 方法二：Schema 合法性检查

```bash
# 如果你安装了 xmllint/libxml2
xmllint --relaxng share/report.rng your_output.xml --noout
```

由于 `report.rng` 已更新，合法的输出报告应能通过 RelaxNG 校验。

---

## 编译后 CLI 用法

拿到新的可执行文件后，直接正常使用即可：

```bash
scram --bdd --probability -o report.xml input_model.xml
```

报告中将只出现 `<bdd-nodes>`，不再有 `<sum-of-products>`。
