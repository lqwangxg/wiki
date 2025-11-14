---
title: 用nssa将bat,ps1包装成windows 服务
description: 
published: true
date: 2025-11-04T05:06:09.230Z
tags: windows, services, nssa
editor: markdown
dateCreated: 2025-11-04T02:50:29.700Z
---


🧩 删除services: 使用 PowerShell 原生命令　
```ps1
$service = Get-Service -Name "filebrowser-imart" -ErrorAction SilentlyContinue
if ($service) {
    Stop-Service -Name "filebrowser-imart" -Force
    sc.exe delete "filebrowser-imart"
    //[SC] DeleteService SUCCESS
    Write-Host "✅ 服务 filebrowser-imart 已删除"
} else {
    Write-Host "⚠️ 未找到服务 filebrowser-imart"
}
```
