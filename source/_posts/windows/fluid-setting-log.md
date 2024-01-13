---
title: windows里设置程序自启动的几种方式
date: 2024-1-13 17:33:55
index_img: /img/index-6.jpg
tags:
  - windows
  - 系统配置
  - jenkins
categories:
  - [windows, 系统配置]
  - jenkins
author: tao-wt@qq.com
excerpt: 总结在之前工作中使用过的，在windows中设置程序自启动的常见的四种方式, 以jenkins slave启动为例
---
## 创建服务(service)
这种方式主要用于windows server版本，可以在`开始`或者`搜索`里查找services来启动services管理器，创建服务。
对于jenkins slave可以通过`javaws`命令来创建:
'''bat
javaws jenkins-slave.jnlp
'''
> 注:
> 1 jenkins-slave.jnlp从jenkins slave详情页面下载
> 2 当服务以本地系统账户运行时可以设置是否能和桌面进行交互

## 创建计划任务(Task Scheduler)
这种方式可以用来指定在系统启动时自动触发运行程序，但这种启动方式程序貌似没法和桌面交互
![Task Scheduler](/img/task_scheduler.png)

## 通过`gpedit.msc`来设置
通过`win + r`, 然后输入`gpedit.msc`来启动`Local Group Policy Editor`进行添加。这种方式程序可以和桌面进行交互。
![gpedit](/img/gpedit.png)

## 通过在startup文件夹创建启动文件来实现
首先，通过`win + r`, 然后输入`shell:startup`来打开**启动**文件夹，然后将要启动的程序或脚本创建一个快捷方式到此文件夹, 即可启用:
jenkins salve的powershell脚本如下：
```powershell
#!powershell

Push-Location -Path "$PSScriptRoot"
while($true)
{
    Start-Process -FilePath "cmd.exe" -Wait `
        -ArgumentList "/c java -jar D:\jenkins\agent.jar -jnlpUrl https://ci.taolife.com/computer/tao%5Ftest%5Fwindows%5F5/jenkins-agent.jnlp -secret 24d727b3180b9c0108637c1ab2a8ebb70bb7061be7816d616a44a6f399c32a551 -workDir d:\jenkins"
    Start-Sleep -Seconds 9
}
```
创建此脚本的快捷方式到`startup`文件夹，然后在快捷方式的属性中可以设置powershell的运行时的参数/属性
`C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -WindowStyle Hidden -File xxx.ps1`
另外，因为这种方式是通过powershell脚本来调bat命令，所以为使powershell脚本能正常启动还应设置powershell `ExecutePolicy`为`remotesigned`:
```powershell
Get-ExecutionPolicy
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```
