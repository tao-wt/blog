---
title: bash脚本多进程编程时对输出日志进行优雅的处理
date: 2024-1-6 20:47:19
index_img: /img/index-5.jpg
tags:
  - linux
  - shell
  - bash
categories:
  - [linux, shell, bash]
author: tao-wt@qq.com
excerpt: 在Shell脚本里运行多个进程时，它们的日志会交错输出在一起，导致日志混乱, 本文提供了一种优雅的处理方式
---
## 简介
日常编写shell脚本时，我们经常需要运行多个进程, 来提升处理的速度。但是，当多个进程同时输出日志时，日志会交错输出在一起，导致日志混乱。本文提供了一种简单优雅的处理方式，可以确保日志可以准确的被区分。

## 实现标记每个进程的输出
为了能区分每个进程的输出日志, 只需要在每行日志前加上进程的信息即可,比如进程的ID。可以写一个简单的shell函数来实现这个功能:
```bash
run_mark_stdout() {
    # Based on the run function, additional information is added to
    # each line of output(program info)
    # to capture function output, use this function carefully
    einfo "Executing: $*"
    local mark="${1}"
    shift 1
    # mark="$(awk '{print $1}' <<< "$*" | awk -F '/' '{print $NF}')"
    "$@" 2>&1 | stdbuf --output=L sed "s/^/[${mark}] /"
}
```
上面函数首先调用传入的命令, 并将标准错误输出重定向到标准输出, 然后通过`sed`命令在每个输出行前加上进程信息。
使用`stdbuf`命令的是因为, 在多个后台进程使用`sed`处理输出时会遇到缓冲区的问题, 导致有的进程输出不是实时的; `stdbuf`命令可以用来改变其后程序的`stdin`, `stdout`和`stderr`的缓冲机制, 例如上面的函数中`stdbuf`命令将`stdout`标准输出的缓冲方式改为了行缓冲, 以保证日志可以被实时输出, 其它两种缓冲方式举例如下:
1. 将后面sed命令的`stderr`缓冲方式设置为全缓冲(fully buffered)
    ```bash
    "$@" 2>&1 | stdbuf --error=5M sed "s/^/[${mark}] /"
    ```
2. 将后面命令的`stdin`缓冲方式设置为无缓冲区
    ```bash
    stdbuf --input=0 commands
    ```
    正常情况, `stdin`一般应为行缓冲

调用以上函数:
```bash
main() {
    mkfifo ./fifo_1.$$ && exec 777<> ./fifo_1.$$ && rm -f ./fifo_1.$$
    mkfifo ./fifo_2.$$ && exec 999<> ./fifo_2.$$ && rm -f ./fifo_2.$$
    pkgs_info=( $(echo "${pkgs_info}" | tr '\n' ' ') )

    set +e
    for pkg_info in "${pkgs_info[@]}"; do
        if grep -q "SYSTEM/" <<< "${pkg_info}"; then
            echo "${pkg_info}" >&999
        else
            echo "${pkg_info}" >&777
        fi
    done
    run_mark_stdout "YANG" process "777" &
    run_mark_stdout "YNC" process "999" &
    ignore_error="no" wait_and_return $(jobs -p)
}
```

输出log如下:
```log
[INFO] >>> Executing: main
[INFO] >>> Executing: YANF process 777
[INFO] >>> Executing: YNC process 999
[INFO] >>> waiting for 1130091
[YANG] [INFO] >>> Variable fd = 777
[YANG] [INFO] >>> Variable artifactory_url = http://8.69.5.82:80/artifactory/api/storage/
[YANG] [INFO] >>> Variable artifactory_user = ****
[YANG] [INFO] >>> Executing: call_download Pro/project/release/,TAO_5
[YNC] [INFO] >>> Variable fd = 999
[YNC] [INFO] >>> Variable artifactory_url = https://test-jfrog.tao.cn/artifactory/api/storage/
[YNC] [INFO] >>> Variable artifactory_user = tao:****
[YNC] [INFO] >>> Executing: call_download /project/****
[YANG] [INFO] >>> Download release...
[YANG] [INFO] >>> Executing: python3 /home/tao/jenkins/workspace/auto_download/downloader.py --uri Yrg_575/Release/ --package USB --target /project/tao/ --unzip_to_usb TAO_YY
[YNC] [INFO] >>> Download ...
```

## 优雅的停止进程
另外, 在一些情况下我们需要自动停止正在运行的程序, 比如在长时间等不到可用资源时跳过等待等待, 可以用`timeout`命令来实现这个功能:
```bash
analyze_with_timeout() {
    local analyze_duration="${1}"
    local kill_after="${2}"
    must_have_var analyze_duration kill_after

    shift 2
    einfo "Executing: $*"
    if [[ "${JOB_NAME}" =~ .*_verify ]]; then
        timeout --kill-after=${kill_after} "${analyze_duration}" "$@" || {
            ewarn "analyze timeout, skip verify."
            exit 0
        }
    else
        "$@"
    fi
}
```
> `timeout`参数说明:
> analyze_duration: kill the command if still running after analyze_duration
> kill_after: also send a **KILL** signal if command is still running this long after the initial signal was sent
