
这份技术文档旨在通过 iPXE 环境实现 Windows 11 的网络自动化安装，并集成驱动、中文语言包以及绕过硬件限制（TPM/Secure Boot）的配置。

* * *

Windows 11 自动化网络安装镜像制作指南
------------------------

### 一、 基础环境准备

1.  **安装工具**：确保已安装 Windows ADK 及 WinPE 加载项。
    
2.  **创建工作目录**：
    
    DOS
    
        copype amd64 D:\winpex
    

### 二、 镜像挂载与驱动集成

在修改镜像前，必须挂载 `boot.wim`。注意 Windows 11 镜像通常包含两个索引，**索引 2** 通常是全功能安装环境。

DOS

    :: 清理之前的挂载状态
    Dism /Cleanup-Wim
    
    :: 挂载 boot.wim 索引 2
    Dism /mount-image /imagefile:D:\winpex\boot.wim /index:2 /mountdir:D:\winpex\mount
    
    :: 注入网卡及存储驱动 (确保 PXE 引导后能识别网络和硬盘)
    dism /image:D:\winpex\mount /add-driver /driver:D:\winpex\windows_driver\ /recurse
    
    :: 导出驱动列表确认
    dism /image:D:\winpex\mount /get-drivers > D:\winpex\boot_drivers.txt

### 三、 中文语言及环境支持

注入必要的语言包和字体支持，防止安装界面出现乱码或默认英文。

DOS

    set ADK_PATH=C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Windows Preinstallation Environment\amd64\WinPE_OCs
    
    :: 1. 核心语言包与字体支持
    dism /Image:D:\winpex\mount /Add-Package /PackagePath:"%ADK_PATH%\zh-cn\lp.cab"
    dism /Image:D:\winpex\mount /Add-Package /PackagePath:"%ADK_PATH%\WinPE-FontSupport-ZH-CN.cab"
    
    :: 2. 安装程序相关组件
    dism /Image:D:\winpex\mount /Add-Package /PackagePath:"%ADK_PATH%\WinPE-Setup.cab"
    dism /Image:D:\winpex\mount /Add-Package /PackagePath:"%ADK_PATH%\zh-cn\WinPE-Setup_zh-cn.cab"
    dism /Image:D:\winpex\mount /Add-Package /PackagePath:"%ADK_PATH%\WinPE-Setup-Client.cab"
    dism /Image:D:\winpex\mount /Add-Package /PackagePath:"%ADK_PATH%\zh-cn\WinPE-Setup-client_zh-cn.cab"
    
    :: 3. 设置全局默认语言为中文
    dism /Image:D:\winpex\mount /Set-AllIntl:zh-cn

### 四、 绕过 Windows 11 硬件检测 (TPM/Secure Boot)

通过离线加载注册表的方式，在 PE 启动前预设跳过检测的逻辑。

DOS

    :: 1. 加载 PE 系统的 SYSTEM 注册表养蜂
    reg load HKLM\WinPE D:\winpex\mount\Windows\System32\config\SYSTEM
    
    :: 2. 写入绕过检测的键值
    reg add "HKLM\WinPE\Setup\LabConfig" /v BypassTPMCheck /t REG_DWORD /d 1 /f
    reg add "HKLM\WinPE\Setup\LabConfig" /v BypassSecureBootCheck /t REG_DWORD /d 1 /f
    reg add "HKLM\WinPE\Setup\LabConfig" /v BypassRAMCheck /t REG_DWORD /d 1 /f
    
    :: 3. 卸载并保存注册表
    reg unload HKLM\WinPE

### 五、 配置自动化启动脚本 (`startnet.cmd`)

修改 `D:\winpex\mount\Windows\System32\startnet.cmd`，实现自动挂载网络共享并启动安装。

代码段

    wpeinit
    :: 延迟 20 秒确保网络栈完全就绪
    ping -n 22 10.13.101.2 > nul
    
    :: 挂载远程共享安装源
    net use z: \\10.13.101.2\install "123456" /user:longsys
    
    :: 进入安装目录并执行
    z:
    cd pxeimage\windows\x64\windows11_25h2_en
    setup.exe

### 六、 保存与生成

DOS

    :: 提交所有更改并卸载镜像
    Dism /unmount-image /mountdir:D:\winpex\mount /commit
    
    :: 生成 ISO 镜像 (可选，通常提取 boot.wim 用于 iPXE)
    MakeWinPEMedia /ISO D:\winpex D:\winpex\winpe_win11_25h2_cn_dis_tpm.iso

* * *

部署建议
----

*   **iPXE 脚本注意**：在引导时，请确保 `initrd` 命令显式重命名文件，例如 `initrd http://.../boot.wim boot.wim`。
    
*   **0xc000000f 预防**：如果报错，请检查 `BCD` 文件内指定的 `osdevice` 路径是否与你上传到 HTTP 服务器的文件路径一致。
    

**您是否需要针对 iPXE 端的 `boot.cfg` 脚本示例来配合这个 WIM 镜像？**

---

