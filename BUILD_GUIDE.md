# SCRAM Windows 编译指南

## 推荐方案：GitHub Actions（无需本地编译环境）

由于你当前机器没有编译环境，我已经帮你写好了 GitHub Actions 工作流。你只需要把代码 push 到 GitHub，CI 会自动编译并产出 `scram.exe`。

### 操作步骤

1. **注册/登录 GitHub**（如果你还没有账号）
   - 访问 https://github.com

2. **创建新仓库**
   - 仓库名随意，例如 `scram-bdd-build`
   - 设置为 Private（私有）或 Public 均可

3. **上传代码**
   在当前项目目录下执行：
   ```bash
   git init
   git add .
   git commit -m "Add BDD node probability output"
   git branch -M main
   git remote add origin https://github.com/你的用户名/scram-bdd-build.git
   git push -u origin main
   ```

   或者直接用 GitHub Desktop / VS Code 上传。

4. **触发编译**
   - 进入 GitHub 仓库页面
   - 点击顶部菜单 **Actions**
   - 左侧选择 **Build SCRAM for Windows**
   - 点击右侧 **Run workflow** → **Run workflow**

5. **下载可执行文件**
   - 等待约 10-15 分钟（安装依赖 + 编译）
   - 编译完成后，进入该 workflow 的运行页面
   - 页面底部 **Artifacts** 区域会显示：
     - `scram-windows-msys2`：仅 `scram.exe` 单文件
     - `scram-windows-full`：完整的 install 目录（含依赖 dll）
   - 点击下载 zip，解压后即可使用

### 运行方式

解压后，在 `scram.exe` 所在目录打开 PowerShell 或 CMD：

```powershell
.\scram.exe --bdd --probability -o report.xml your_model.xml
```

> 注意：MSYS2 编译的版本需要一些运行时 DLL（如 `libboost_program_options-*.dll`、`libxml2-*.dll`、`libgcc_s_seh-1.dll` 等），建议下载 **scram-windows-full**  artifact，里面包含了所有需要的依赖。

---

## 本地编译方案（需配置环境）

如果你以后想在本机编译，以下是 Windows 上最可靠的步骤（与项目官方 CI 一致）：

### 步骤 1：安装 MSYS2

1. 下载 MSYS2 安装器：https://www.msys2.org/
2. 运行安装器，按默认选项安装到 `C:\msys64`
3. 安装完成后，打开 **MSYS2 MINGW64** 终端（开始菜单里找）

### 步骤 2：安装编译依赖

在 MSYS2 MINGW64 终端中执行：

```bash
pacman -Syu
pacman -S --needed \
  mingw-w64-x86_64-gcc \
  mingw-w64-x86_64-cmake \
  mingw-w64-x86_64-make \
  mingw-w64-x86_64-boost \
  mingw-w64-x86_64-libxml2
```

### 步骤 3：编译 SCRAM

```bash
cd /c/Users/admin/Downloads/scram-develop  # 替换为你的实际路径

mkdir build
cd build

cmake .. -G "MSYS Makefiles" \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=../install \
  -DBUILD_TESTING=ON

cmake --build . --parallel $(nproc)

# 安装（可选）
cmake --build . --target install/strip
```

### 步骤 4：运行

```bash
./install/bin/scram --bdd --probability -o report.xml ../input/TwoTrain/two_train.xml
```

---

## Docker 方案（需要 Windows 专业版/企业版）

Windows 上安装 Docker Desktop 需要：
- Windows 10/11 专业版、企业版或教育版（家庭版不支持 WSL2/Hyper-V）
- 在 BIOS 中开启虚拟化

### 安装步骤

1. **开启 WSL2**（管理员 PowerShell）：
   ```powershell
   wsl --install
   ```

2. **下载安装 Docker Desktop**：
   - 访问 https://www.docker.com/products/docker-desktop/
   - 下载并运行安装程序
   - 安装过程中勾选 **Use WSL 2 instead of Hyper-V**

3. **验证安装**：
   ```powershell
   docker --version
   ```

4. **编译 SCRAM**：
   在项目目录下执行：
   ```powershell
   docker build -t scram .
   ```
   这会自动使用项目自带的 `Dockerfile` 编译。

5. **提取可执行文件**：
   由于 Docker 容器内编译的二进制是 Linux ELF 格式，**不能在 Windows 上直接运行**。因此 Docker 方案在 Windows 上主要用于验证/测试，如果你想拿到 Windows 原生 `.exe`，请使用上面的 **GitHub Actions** 或 **MSYS2 本地编译** 方案。

---

## 为什么当前环境无法直接安装？

你的当前环境：
- 已安装：Git for Windows、Python、winget
- **未安装**：MSYS2、MinGW GCC、CMake、Docker

`winget` 在自动化脚本中无法处理交互式许可协议确认，因此自动安装失败。MSYS2、CMake、GCC 都需要通过安装包或交互式命令安装，无法在无交互权限的脚本中一键完成。

**因此最推荐的路径是：GitHub Actions（你什么都不用装，等 15 分钟下载 exe 即可）。**
