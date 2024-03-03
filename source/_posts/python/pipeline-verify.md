---
title: 利用asyncio实现高效的CI脚本门禁校验功能
date: 2024-3-3 16:15:52
index_img: /img/index-13.jpg
tags:
  - python
categories:
  - [python, asyncio]
author: tao-wt@qq.com
excerpt: 利用python的asyncio模块来实现高效的Jenkinsfile和Python脚本的语法检查小脚本
---
## 背景
最近要实现CI代码的门禁功能，由于CI脚本是使用gerrit进行版本管理的，所以门禁功能基于gerrit的change review机制来实现(只有`Code Review +2`和`Verify +1`后才允许change合入)，而对于代码门禁最重要的是校验速度。
工具选择：
- python代码选用Flake8进行校验，其相对pylint来说更轻量级。
- 因为是声明式的pipeline脚本，所以使用jenkins自带的lint工具进项校验，通过调用jenkins API实现

另外，由于整个校验的过程数据的处理不算多，并且也涉及到了大量的IO操作，所以采用asyncio异步编程来提升门禁的速度。
> 注：本文不涉及gerrit的配置和jenkins job的触发，不包括运行环境的配置，也不涉及怎么获取到要校验的change到工作目录

## 脚本
此脚本首先遍历和过滤出要校验的脚本文件，然后对每一个要校验的文件创建一个异步`asyncio.Task`任务放入事件循环中执行，最后根据检查出来的问题通过gerrit API对gerrit change进行review，如：在出问题的行上进行评论和`verify`标签的`+/-1`。
```python
import os
import re
import asyncio
import aiohttp
import aiofiles
from utils import logger


DEBUG = True if os.getenv("Debug", "") == "true" else False
log = logger.setup_logger(debug=DEBUG)
RESULTS = dict()
COMMENTS = dict()
JENKINS_URL = os.environ["JENKINS_URL"].rstrip("/")
GERRIT_URL = os.environ["GERRIT_URL"].rstrip("/")
CI_USER = os.environ["ci_user"]
CI_PASSWD = os.environ["ci_passwd"]


async def run_proc(cmd: str) -> asyncio.subprocess.Process:
    proc = await asyncio.create_subprocess_shell(
        cmd,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.STDOUT
    )
    return proc


async def get_file_list():
    cmd = "git diff --name-status HEAD^!"
    proc = await run_proc(cmd)
    while line := await proc.stdout.readline():
        line = line.decode().strip()
        log.debug(line)
        yield line
    if await proc.wait() != 0:
        raise Exception(f"'{cmd}' failed!")


async def parse_git_show(state: str, file: str) -> list:
    cmd = f"git show HEAD {file}"
    proc = await run_proc(cmd)
    stdout, _ = await proc.communicate()
    if proc.returncode != 0:
        log.error(f"'{cmd}' failed: {stdout}")
        raise Exception(f"'{cmd}' failed")

    linenos = list()
    flag = False
    for line in stdout.decode().split("\n"):
        log.debug(line)
        if not flag:
            if match_obj := re.match(r"@@\s-\d+,\d+\s\+(\d+),(\d+)", line):
                flag = True
                lineno = int(match_obj.group(1))
            continue
        if re.match(r"-", line):
            continue
        if re.match(r"\+", line):
            linenos.append(lineno)
        lineno += 1
        if lineno > int(match_obj.group(1)) + int(match_obj.group(2)) - 1:
            flag = False
    return linenos


async def flake8(file) -> dict:
    cmd = f"flake8 {file}"
    proc = await run_proc(cmd)
    stdout, _ = await proc.communicate()
    log.debug(f"proc.returncode: {proc.returncode}")
    if proc.returncode == 0:
        log.info(f"'{file}' flake8 check pass")
        return dict()

    result = dict()
    for line in stdout.decode().split("\n"):
        if not line:
            continue
        if match_obj := re.match(f"{file}:(\\d+):", line):
            lineno = int(match_obj.group(1))
            if lineno in result:
                result[lineno] = result[lineno] + f"\n{line}"
            else:
                result[lineno] = line
        else:
            log.warning(f"line not match: '{line}'")
    return result


def comment(file: str, line: int, info: str) -> str:
    if file not in COMMENTS:
        COMMENTS[file] = list()
    COMMENTS[file].append({"line": line, "message": info})
    return info


async def verify_python(state, file):
    async with asyncio.TaskGroup() as tg:
        linenos = await tg.create_task(parse_git_show(state, file))
        out = await tg.create_task(flake8(file))

    log.debug(f"linenos: {linenos!r}")
    log.debug(f"flake8_out: {out!r}")
    lines = [comment(file, no, out[no]) for no in linenos if no in out]
    if lines:
        RESULTS[file] = "\n".join(lines)


async def file_sender(file_name=None):
    async with aiofiles.open(file_name, 'rb') as f:
        chunk = await f.read(64*1024)
        while chunk:
            yield chunk
            chunk = await f.read(64*1024)


async def verify_jenkinsfile(jenkins, file: str) -> None:
    uri = "/pipeline-model-converter/validate"
    data = aiohttp.FormData()
    data.add_field(
        'jenkinsfile',
        file_sender(file),
        content_type='text/plain; charset=utf-8'
    )

    async with jenkins.post(uri, data=data) as resp:
        assert resp.status == 200, \
            f"post {uri} failed! error: {await resp.text()}"
        resp_text = await resp.text()
        if "successfully validated" not in resp_text:
            if re_obj := re.search(r"@\sline\s(\d+),\scolumn", resp_text):
                comment(file, int(re_obj.group(1)), resp_text)
            RESULTS[file] = resp_text


async def comment_change(auth: aiohttp.BasicAuth) -> None:
    change_id = os.environ["GERRIT_CHANGE_ID"]
    revision = os.environ["GERRIT_PATCHSET_REVISION"]
    uri = f"/a/changes/{change_id}/revisions/{revision}/review"
    data = {"message": "Change verify failed!!!", "comments": COMMENTS}

    async with aiohttp.ClientSession(GERRIT_URL, auth=auth) as gerrit:
        async with gerrit.post(uri, json=data) as resp:
            assert resp.status == 200, \
                f"post {uri} failed! error: {await resp.text()}"
            log.info("comment to change succeed")


async def check_result(auth: aiohttp.BasicAuth) -> int:
    if not RESULTS:
        log.info("change passed the verify")
        return 0

    msg = [f"***** {file} *****\n{RESULTS[file]}" for file in RESULTS]
    msg = "Change verify failed:\n" + "\n\n".join(msg)
    log.info(msg)
    await comment_change(auth)
    return 1


async def main() -> int:
    tasks = list()
    auth = aiohttp.BasicAuth(CI_USER, CI_PASSWD)
    async with (
        aiohttp.ClientSession(JENKINS_URL, auth=auth) as jenkins,
        asyncio.TaskGroup() as tg
    ):
        async for line in get_file_list():
            if re.match(r"[MA]\spython-scripts/.*\.py$", line):
                tasks.append(tg.create_task(verify_python(*line.split())))
            elif re.match(r"[MA]\sjenkinsfiles/", line):
                tasks.append(tg.create_task(
                    verify_jenkinsfile(jenkins, line.split()[1])
                ))
            else:
                log.info(f"'{line}' does not match with any regex str.")

    return await check_result(auth)


if __name__ == '__main__':
    if os.name == 'posix':
        import uvloop
        asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())
    exit(asyncio.run(main()))

```
