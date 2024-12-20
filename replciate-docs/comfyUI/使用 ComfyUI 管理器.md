### 使用 ComfyUI 管理器

### 1. 安装 ComfyUI 管理器

在使用示例之前，您需要先安装由 **ltdrdata** 提供的 ComfyUI 管理器。这是一个自定义节点，能够帮助您轻松管理所有的 ComfyUI 自定义节点和权重文件。

#### **安装步骤**

最简单的安装方式是将 ComfyUI-Manager 存储库克隆到您的自定义节点目录中：`ComfyUI/custom_nodes`。

**执行以下命令：**
```bash
cd ComfyUI/custom_nodes
git clone https://github.com/ltdrdata/ComfyUI-Manager.git
```
执行完成后，您会在路径 `ComfyUI/custom_nodes/ComfyUI-Manager` 下看到一个新的目录。

> **注意：** 如果您使用的是 Linux 系统，可能需要额外的安装步骤。有关更多详细信息，请参阅 ComfyUI-Manager 的 [README](https://github.com/ltdrdata/ComfyUI-Manager) 文件。

---

### 2. 开始安装模型和节点

成功安装 ComfyUI 管理器后，您将可以更加方便地安装模型和节点。

#### **可用的功能**
- **安装自定义节点**：一键安装所需的自定义节点。
- **安装缺失的自定义节点**：当您使用他人的工作流程时，这个功能尤其有用，它会自动为您安装缺失的节点。
- **安装模型**：支持安装检查点（checkpoints）、LoRA 模型等各种类型的模型。

#### **其他功能**
- **更新所有自定义节点**：一键更新已安装的所有自定义节点。
- **更新 ComfyUI**：保持您的 ComfyUI 版本始终为最新版本。

