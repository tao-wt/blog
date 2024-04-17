---
title: 华为对象存储OBS异步上传客户端
date: 2024-4-16 19:40:52
index_img: /img/index-9.jpg
tags:
  - python
categories:
  - [python, asyncio]
author: tao-wt@qq.com
excerpt: 利用python的asyncio模块来实现异步上传文件到华为对象存储
---
## 背景
最近工作中需要收集测试报告及运行日志上传到华为对象存储OBS，本来打算用华为OBS自带的python SDK来上传，由于其它的代码都是异步的，同步的OBS上传代码怎么看怎么别扭，所以就看了下华为的对象存储API，用异步的方式重新实现了文件上传功能（包括多段上传），主要功能如下：
- 当文件大于100M时自动改用分段上传，提升上传速度，当分段上传失败会自动回退删除已创建的多段任务
- 自动判断文件类型(HTTP `Content-Type`)和`charset`，方便后续开启OBS网站静态托管
- 默认并发上传数量限制为20

> 注：
> 文件的`x-obs-acl`固定设置为`public-read`
> 本模块运行需要特定环境变量, 并需要传入`aiohttp.ClientSession`对象；后面如果有时间的话，我会对这块进行优化，写成一个功能完整的异步模块
> 本模块暂不支持递归对目录下文件进行上传，需在调用代码实现目录文件的遍历
> 本模块暂不支持断点续传上传

## 模块使用
1. 首先设置执行环境
    ```powershell
    PS D:\Code>
    PS D:\Code> $env:obs_domain = "obsv3.zj-jko-1.tao-wt.com"
    PS D:\Code> $env:obs_bucket = "lk-we-ci-tao-prod-oss"
    PS D:\Code> $env:OBS_SECRET_KEY = "S3DespvWdoR9c54DqEDoE462xzaEtYaDEI908iHOX"
    PS D:\Code> $env:OBS_ACCESS_KEY = "KSV5FPS7U000AFGZVEFY"
    ```

2. 创建aiohttp session客户端， 并导入本模块, 本文以python REPL交互式环境为例。(正式使用时，推荐在`async with`方式在上下文管理器中使用)
    ```python
    >>> import asyncio
    >>> import os
    >>> import aiohttp
    >>> from utils import async_obs
    >>> OBS_DOMAIN = os.environ['obs_domain']
    >>> OBS_BUCKET = os.environ['obs_bucket']
    >>> OBS_HOST = f"{OBS_BUCKET}.{OBS_DOMAIN}"
    >>> OBS_BASE_URL = f"https://{OBS_HOST}"
    >>> # 创建异步的session client, SSL证书校验建议打开
    >>> obs = aiohttp.ClientSession(OBS_BASE_URL, connector=aiohttp.TCPConnector(ssl=False))
    ```

3. 上传小的文本文件, 上传完成后自动返回对象连接
    ```python
    >>> url = await async_obs.upload(obs, "d:\\usb_count.txt", upload_to="/tao_test/usb_count.txt")
    >>> url
    'https://lk-we-ci-tao-prod-oss.obsv3.zj-jko-1.tao-wt.com/tao_test/usb_count.txt'
    >>>
    ```

4. 上传大文件，会自动采用分段上传
    ```python
    >>>
    >>> large_fileurl = await async_obs.upload(obs, "d:\\pkg_test.7z", upload_to="/tao_test/pkg_test.7z")
    [INFO] async_obs@234: uploadId: 9400358EE6F39C1BBC0931F123667330
    >>> large_fileurl
    'https://lk-we-ci-tao-prod-oss.obsv3.zj-jko-1.tao-wt.com/tao_test/pkg_test.7z'
    >>>
    ```
    为了验证上传的完整性：从OBS上下载上传的文件并和原始文件的hash进行比较，两者一样。(这一步是验证性步骤，可忽略)
    ```powershell
    PS C:\Users\Tao\Downloads> Get-FileHash .\pkg_test.7z

    Algorithm       Hash                                                                   Path
    ---------       ----                                                                   ----
    SHA256          5EB361807CE3208484005FEB82D6103C874C4B00AD549DD3DC093A405B72A821       C:\Users\Tao\Downloads...


    PS C:\Users\Tao\Downloads> Get-FileHash D:\pkg_test.7z

    Algorithm       Hash                                                                   Path
    ---------       ----                                                                   ----
    SHA256          5EB361807CE3208484005FEB82D6103C874C4B00AD549DD3DC093A405B72A821       D:\pkg_test.7z


    PS C:\Users\Tao\Downloads>
    ```

5. 使用完成后关闭session对象，（推荐以`async with`方式使用）
    ```python
    >>> obs.close()
    <coroutine object ClientSession.close at 0x000001E57F4AE080>
    >>> await obs.close()
    >>>
    ```


## 脚本
### logger模块
logger模块如下：
```python
import logging
import logging.handlers


def get_formatter():
    prefix_format = [
        "[%(levelname)s]",
        " %(module)s@%(lineno)s: "
    ]
    return ''.join(prefix_format) + "%(message)s"


def get_handler(sub_proc_queue, debug):
    if sub_proc_queue:
        handler = logging.handlers.QueueHandler(sub_proc_queue)
    else:
        handler = logging.StreamHandler()
    if debug:
        handler.setLevel(logging.DEBUG)
    else:
        handler.setLevel(logging.INFO)
    return handler


def init_logger(name, propagate, empty_root_handlers):
    if empty_root_handlers:
        logging.getLogger().handlers = list()
    logger = logging.getLogger(name)
    logger.setLevel(logging.DEBUG)
    logger.propagate = propagate
    return logger


def setup_logger(name="root", *, debug=False, handlers=list(),
                 save_to_file=False, sub_proc_queue=None, format_str='',
                 propagate=True, empty_root_handlers=False):
    logger = init_logger(name, propagate, empty_root_handlers)
    sh = get_handler(sub_proc_queue, debug)
    handlers.append(sh)
    if not sub_proc_queue:
        if not format_str:
            format_str = get_formatter()
        sh.setFormatter(logging.Formatter(format_str))
        if name and save_to_file:
            fh = logging.FileHandler(name + ".log")
            fh.setLevel(logging.DEBUG)
            fh.setFormatter(logging.Formatter(format_str))
            logger.addHandler(fh)
            handlers.append(fh)
    logger.addHandler(sh)
    return logger

```

### async_obs模块
async_obs.py代码如下，运行前需要安装aiohttp, aiofiles模块：
```python
import os
import base64
import re
import binascii
import chardet
import asyncio
import contextvars
import hashlib
import hmac
import aiohttp
from datetime import datetime, timezone
import mimetypes
import aiofiles
from . import logger


log = logger.setup_logger(__name__)
OBS_DOMAIN = os.environ['obs_domain']
OBS_BUCKET = os.environ['obs_bucket']
OBS_SECRET_KEY = os.environ['OBS_SECRET_KEY']
OBS_ACCESS_KEY = os.environ['OBS_ACCESS_KEY']
OBS_HOST = f"{OBS_BUCKET}.{OBS_DOMAIN}"
OBS_BASE_URL = f"https://{OBS_HOST}"
FILE_PATH = contextvars.ContextVar("file path")
UPLOAD_TO = contextvars.ContextVar("OBS object", default=None)
MIME_TYPE = contextvars.ContextVar("mime type")
PARTS = contextvars.ContextVar("parts", default=dict())
OBS_SESSION = contextvars.ContextVar("obs session")
MAX_LENGTH = 100 * 1024 * 1024
SEM = asyncio.Semaphore(20)


def get_time() -> str:
    now = datetime.now(tz=timezone.utc)
    return now.strftime("%a, %d %b %Y %H:%M:%S %Z")


def get_authorization(method, headers, uri) -> str:
    canonicalizedheaders = get_canonicalizedheaders(headers)
    canonical_list = [
        method,
        headers.get('Content-MD5', ""),
        headers.get('Content-Type', ""),
        headers.get('Date', ""),
        canonicalizedheaders + f'/{OBS_BUCKET}{uri}'
    ]

    hashed = hmac.new(
        OBS_SECRET_KEY.encode('UTF-8'),
        "\n".join(canonical_list).encode('UTF-8'),
        hashlib.sha1
    )
    encode_canonical = binascii.b2a_base64(
        hashed.digest()
    )[:-1].decode('UTF-8')
    return f"OBS {OBS_ACCESS_KEY}:{encode_canonical}"


def get_canonicalizedheaders(headers: dict) -> str:
    result = ''
    for key in sorted(headers):
        key = key.lower()
        if key.startswith('x-obs-'):
            if any(ord(c) > 127 for c in key):
                raise Exception("header's key contain non-ascii charactor!")
            result += f"{key}:{headers[key]}\n"
    return result


async def get_rfc1864_md5(start=0, end=None, byte_str: bytes = None):
    hasher = hashlib.md5()
    if byte_str:
        hasher.update(byte_str)
    else:
        async for data in file_sender(start, end):
            hasher.update(data)
    md5_value = hasher.digest()
    base64_md5_value = base64.b64encode(md5_value).decode('utf-8')
    return base64_md5_value


async def get_encoding(file):
    async with aiofiles.open(file, mode='rb') as fd:
        data = await fd.read()
    return chardet.detect(data)['encoding']


async def get_mimetype():
    file_path = FILE_PATH.get()
    content_type, _ = mimetypes.guess_type(file_path)
    if not content_type:
        content_type = 'application/octet-stream'
    elif file_path.endswith('.txt'):
        charset = await get_encoding(file_path)
        content_type = f"{content_type};charset={charset}"
    return content_type


async def file_sender(start=0, end=None):
    length = 64*1024
    async with aiofiles.open(FILE_PATH.get(), 'rb') as f:
        if start:
            await f.seek(start)
        while True:
            if end:
                tell = await f.tell()
                assert tell <= end, "read too much data!"
                if end == tell:
                    break
                left = end - tell
                if length > left:
                    length = left
            data = await f.read(length)
            if not data:
                break
            yield data


def generate_objectname(upload_to=None):
    if not upload_to:
        job = os.environ['JOB_NAME']
        build = os.environ['BUILD_ID']
        file_name = os.path.basename(FILE_PATH.get())
        upload_to = f"/{job}/{build}/{file_name}"
    return upload_to


async def put(size, start: int = None, end: int = None, params: dict = {}):
    uri = UPLOAD_TO.get()
    if params:
        uri = uri + "?" + "&".join([f"{k}={params[k]}" for k in params])
    mime = 'application/octet-stream' if start else MIME_TYPE.get()
    headers = {
        'Host': OBS_HOST,
        'Content-Length': str(size),
        'x-obs-acl': 'public-read',
        'Date': get_time(),
        'Content-Type': mime,
        'Content-MD5': await get_rfc1864_md5(start, end)
    }
    headers['Authorization'] = get_authorization("PUT", headers, uri)

    OBS = OBS_SESSION.get()
    async with (
        SEM,
        OBS.put(uri, data=file_sender(start, end), headers=headers) as resp
    ):
        assert resp.status == 200, \
            f"upload {uri} failed! error: {await resp.text()}"
        if start is not None:
            parts = PARTS.get()
            parts[params["partNumber"]] = resp.headers['ETag']
            PARTS.set(parts)
    return f"{OBS_BASE_URL}{UPLOAD_TO.get()}"


async def del_uploads(upload_id):
    uri = f"{UPLOAD_TO.get()}?uploadId={upload_id}"
    headers = {'Host': OBS_HOST, 'Date': get_time()}
    headers['Authorization'] = get_authorization("DELETE", headers, uri)

    async with OBS_SESSION.get().delete(uri, headers=headers) as resp:
        assert resp.status == 204, \
            f"delete uploads {upload_id} failed: {await resp.text()}"


def split_file_yield(size):
    step = MAX_LENGTH
    for offset in range(0, size, step):
        start = offset
        left = size - offset
        end, length = (offset + step, step) if left > step else (size, left)
        yield (start, end, length)


async def init_uploads():
    uri = f"{UPLOAD_TO.get()}?uploads"
    headers = {
        'Host': OBS_HOST,
        'Content-Type': MIME_TYPE.get(),
        'x-obs-acl': 'public-read',
        'Date': get_time()
    }
    headers['Authorization'] = get_authorization("POST", headers, uri)
    async with OBS_SESSION.get().post(uri, headers=headers) as resp:
        assert resp.status == 200, \
            f"init uploads {uri} failed! error: {await resp.text()}"
        result = await resp.text()
    return re.search(r'UploadId>([^<\s]+)</UploadId', result).group(1)


async def puts(size, upload_id):
    part_number = 1
    tasks = set()
    async with asyncio.TaskGroup() as tg:
        for start, end, length in split_file_yield(size):
            params = {"partNumber": part_number, "uploadId": upload_id}
            tasks.add(tg.create_task(put(length, start, end, params)))
            part_number += 1


async def merge_body() -> bytes:
    parts = PARTS.get()
    log.debug(parts)
    sorted_key = sorted(parts)
    part = "<Part><PartNumber>{}</PartNumber><ETag>{}</ETag></Part>"
    body = ''.join([part.format(key, parts[key]) for key in sorted_key])
    body = f"<CompleteMultipartUpload>{body}</CompleteMultipartUpload>"
    log.debug(body)
    bs = body.encode()
    return bs, await get_rfc1864_md5(byte_str=bs)


async def merge(upload_id):
    uri = f"{UPLOAD_TO.get()}?uploadId={upload_id}"
    body, md5 = await merge_body()
    headers = {
        'Host': OBS_HOST,
        'Content-Length': str(len(body)),
        'Date': get_time(),
        'Content-MD5': md5,
        'Content-Type': 'application/xml;charset=utf-8',
    }
    headers['Authorization'] = get_authorization("POST", headers, uri)

    OBS = OBS_SESSION.get()
    async with OBS.post(uri, headers=headers, data=body) as resp:
        assert resp.status in [200, 500, 503], \
            f"init uploads {uri} failed! error: {await resp.text()}"


async def chunked_put(size: int) -> str:
    upload_id = await init_uploads()
    log.info(f"uploadId: {upload_id}")

    try:
        await puts(size, upload_id)
    except Exception:
        log.error(f"call 'puts({size}, {upload_id})' failed.")
        await del_uploads(upload_id)
        raise

    log.debug("finish upload all parts of the file")
    await merge(upload_id)
    log.debug("finish merge all parts of the file")
    return f"{OBS_BASE_URL}{UPLOAD_TO.get()}"


async def upload(
    session: aiohttp.ClientSession,
    file_path: str, *, upload_to: str = None
) -> str:
    """
    @session: aiohttp.ClientSession, obs aiohttp client session
    @file_path: str, the local file path
    @upload_to: str, the remote path to store,
        default is: /JOB_NAME/BUILD_ID/basename(file_path)
    return the file download url
    """
    FILE_PATH.set(file_path)
    UPLOAD_TO.set(generate_objectname(upload_to))
    MIME_TYPE.set(await get_mimetype())
    OBS_SESSION.set(session)

    size = os.path.getsize(file_path)
    if MAX_LENGTH < size:
        url = await chunked_put(size)
    else:
        url = await asyncio.create_task(put(size))
    return url

```
