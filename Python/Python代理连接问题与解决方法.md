# Python 下载模型时的代理连接问题与解决方法

## 1. 遇到了什么问题？

运行 UNetFormer 时，程序尝试从 Hugging Face 下载 ResNet18 预训练权重：

```text
https://huggingface.co/timm/resnet18.fb_swsl_ig1b_ft_in1k/resolve/main/model.safetensors
```

下载失败，出现类似报错：

```text
requests.exceptions.ProxyError
Unable to connect to proxy
SSLError: TLS/SSL connection has been closed (EOF)
```

这说明 **Python 发起 HTTPS 请求时，代理连接出现了问题**。常见原因是终端设置了无效的代理地址或端口、代理软件未运行，或者代理与目标网站之间的 TLS 连接中断。它不意味着 UNetFormer 模型代码有问题。

## 2. 怎么解决？

以下操作以 **Windows CMD / Anaconda Prompt** 为例。

### 第一步：查看是否配置了代理

```bat
echo %HTTP_PROXY%
echo %HTTPS_PROXY%
```

如果输出类似 `http://127.0.0.1:7890`，表示当前终端配置了代理。`7890` 只是示例端口。

### 第二步：根据网络情况处理代理

**情况 A：当前网络可以不通过代理访问 Hugging Face**

清除当前终端的代理环境变量：

```bat
set HTTP_PROXY=
set HTTPS_PROXY=
set ALL_PROXY=
set http_proxy=
set https_proxy=
set all_proxy=
```

这些命令只对**当前 CMD / Anaconda Prompt 窗口**生效，不会永久更改系统配置。

**情况 B：必须通过代理才能访问 Hugging Face**

先确认代理软件正在运行，并找到它实际使用的 **HTTP 代理端口**。例如端口是 `7890`：

```bat
set HTTP_PROXY=http://127.0.0.1:7890
set HTTPS_PROXY=http://127.0.0.1:7890
```

注意：将 `7890` 替换为真实端口；`HTTPS_PROXY` 的值使用 `http://` 是常见的 HTTP 代理配置方式，不代表目标网站不是 HTTPS。

### 第三步：验证是否恢复连接

在同一终端执行：

```bat
python -c "import requests; r = requests.get('https://huggingface.co', timeout=15); print(r.status_code)"
```

- 返回 `200`：网站访问成功，可以重新运行训练命令，检查权重能否正常下载。
- 仍提示 `ProxyError`：检查代理软件、端口，以及是否还有其他代理配置。
- 提示 `Timeout`、`ConnectionError`：可能是当前网络无法直连，需要使用可用的代理。

## 3. 总结

遇到 Python 下载依赖或模型时出现 `ProxyError`，优先检查**当前终端的代理环境变量**：

**查看代理 → 直连可用则清除代理 / 需要代理则设为正确地址 → 测试连接 → 重试下载。**

不要将 `pretrained=False` 当作网络问题的首选修复方法，因为这会改变模型的训练初始化方式。
