---
layout: post
title: Open WebUI安装
date: 2026-01-20 +0800
tags: ["AI"]
---


# Open WebUI安装
> `Open WebUI` 是 Ollama 的 Web 客户端。`docker run` 启动服务，访问 `http://localhost:3000/`。

<!-- more -->
## docker安装Open WebUI
```shell

# 从镜像中心拉取open-webui镜像
docker pull ghcr.io/open-webui/open-webui:main

# docker run启动open webui服务，目前ollama和open webui部署在同一台机器上
docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway \
  -v open-webui:/app/backend/data \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

## 页面访问
`Open WebUI`访问地址 http://localhost:3000/, 当`Ollama`中部署有多个模型时，可在左上角切换模型名称。
<div>
    <img src="/resource/2026/ai/open-webui.jpg" width="590"/>
</div>