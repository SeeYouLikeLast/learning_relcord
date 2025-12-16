在 Ubuntu 系统之间迁移 VS Code 的插件和配置，主要有两种最常用的方法。

**强烈推荐方法一（使用账号同步）**，这是官方原生功能，最省心且不易出错。如果你因网络或其他原因无法登录账号，请使用**方法二（命令行导出/导入）**。

-----

### 方法一：使用 VS Code 自带的“设置同步” (推荐)

这是最简单的方法，它会自动同步你的插件、配置 (`settings.json`)、快捷键和代码片段。

1.  **在旧电脑上：**

      * 打开 VS Code。
      * 点击左下角的**齿轮图标** (管理) -\> 选择 **Turn on Settings Sync... (开启设置同步)**。
      * 点击 **Sign in & Turn on**，选择使用 GitHub 或 Microsoft 账号登录。
      * 勾选所有选项（Settings, Keyboard Shortcuts, Extensions 等）。
      * 等待上传完成。

2.  **在新电脑上：**

      * 打开 VS Code。
      * 同样点击左下角齿轮 -\> **Turn on Settings Sync...**。
      * 登录同一个账号。
      * VS Code 会自动下载并安装所有插件和配置。

-----

### 方法二：命令行手动导出与导入 (离线/无账号方案)

如果你需要生成一个文件列表并通过 U 盘或局域网传输，请按以下步骤操作：

#### 第一步：在旧电脑导出插件列表

1.  打开终端 (`Ctrl+Alt+T`)。
2.  运行以下命令，将当前安装的所有插件 ID 导出到一个文本文件中：
    ```bash
    code --list-extensions > vscode-extensions.txt
    ```
3.  这会在你当前的目录下生成一个名为 `vscode-extensions.txt` 的文件。

#### 第二步：备份配置文件 (Settings & Keybindings)

仅仅导出插件列表是不够的，你还需要备份你的个性化设置（字体大小、主题配置等）。

1.  进入 VS Code 的用户配置目录：
    ```bash
    cd ~/.config/Code/User/
    ```
2.  你需要复制以下关键文件（如果存在）：
      * `settings.json` (核心配置文件)
      * `keybindings.json` (快捷键配置)
      * `snippets/` (文件夹，存放代码片段)
3.  你可以把它们和上面的 `vscode-extensions.txt` 一起打包：
    ```bash
    # 假设你还在生成 txt 的那个目录，创建一个备份文件夹
    mkdir vscode-backup
    cp vscode-extensions.txt vscode-backup/
    cp ~/.config/Code/User/settings.json vscode-backup/
    cp ~/.config/Code/User/keybindings.json vscode-backup/ 2>/dev/null
    cp -r ~/.config/Code/User/snippets/ vscode-backup/ 2>/dev/null

    # 压缩以便传输
    tar -czvf vscode-backup.tar.gz vscode-backup
    ```

#### 第三步：在新电脑导入

1.  **传输文件：** 将 `vscode-backup.tar.gz` 复制到新电脑并解压。

2.  **批量安装插件：**
    打开终端，进入解压后的目录，运行以下脚本批量安装插件：

    ```bash
    cat vscode-extensions.txt | xargs -L 1 code --install-extension
    ```

    *(解释：这行命令会读取文本中的每一行插件ID，并自动执行安装命令)*

3.  **恢复配置文件：**
    将配置文件覆盖到新电脑的配置目录中：

    ```bash
    cp settings.json ~/.config/Code/User/
    cp keybindings.json ~/.config/Code/User/ 2>/dev/null
    cp -r snippets/ ~/.config/Code/User/ 2>/dev/null
    ```

-----

### 特别提示

  * **服务器端插件：** 如果你经常使用 `Remote - SSH` 连接服务器开发，远程服务器上的插件需要在连接后单独安装（通常 VS Code 会提示你在远程安装）。
  * **大文件环境：** C++ (IntelliSense) 或 Python 等插件安装后，可能需要重新下载对应的 Language Server 二进制文件，这取决于网络情况。

**建议：** 先尝试**方法一**，如果不行再用**方法二**。
