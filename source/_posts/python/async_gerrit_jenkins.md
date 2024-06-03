---
title: 初步的不完善的gerrit,confluence和jenkins异步客户端
date: 2024-5-28 15:26:52
index_img: /img/index-17.jpg
tags:
  - python
  - gerrit
  - jenkins
  - aiohttp
categories:
  - [python, aiohttp]
  - jenkins
  - gerrit
author: tao-wt@qq.com
excerpt: 这两个模块只是满足了最近工作中的需求，还未仔细雕琢，等以后有时间再做扩展和优化
---
# Gerrit模块
目前已实现了以下功能：
- 列出项目分支，分支的创建，获取分支信息以及分支下文件内容
- 获取某个commit下的改动的文件列表，文件内容
- 创建change，修改/添加/删除文件，review change和提交change

## 示例

### 初始化
首先创建`aGerrit`实例，获取`aProject`对象：
```python
>>> gerrit = aGerrit(GERRIT_URL, USER, PASSWD)
>>> project = gerrit.project("tao-wt/pipeline")
```

### 分支操作
获取project所有分支信息，和指定分支信息：
```python
>>> branches = await project.list_branches()
>>> branches
'[{"ref":"HEAD","revision":"master"},{"web_links":[{"name":"browse","url":"/plugins/gitiles/tao-wt/pipeline/+/refs/meta/config","target":"_blank"}],"ref":"refs/meta/config","revision":"997a28a6352656c6b25e7ff5344f2a905c37b1d5"},{"web_links":[{"name":"browse","url":"/plugins/gitiles/tao-wt/pipeline/+/refs/heads/master","target":"_blank"}],"ref":"refs/heads/master","revision":"99070d59a721d1ad01a20457cd34b80312698709"}]\n'
>>> await project.get_branch("master")
'{"web_links":[{"name":"browse","url":"/plugins/gitiles/tao-wt/pipeline/+/refs/heads/master","target":"_blank"}],"ref":"refs/heads/master","revision":"99070d59a721d1ad01a20457cd34b80312698709","can_delete":true}\n'
>>>
```

获取`master`分支`.flake8`文件内容：
```python
>>> await project.get_content("master", ".flake8")
'[flake8]\nignore =\n    # import at the begining of a module is customary but not required\n    E402\nmax-line-length = 100\n'
>>>
```

### commit操作
获取指定commit信息，并通过commit获取文件内容：
```python
>>> commit = project.commit()
>>> await commit.get("59a9264ce673baa992e84b35c31ea079f1873d1f")
'{"commit":"59a9264ce673baa992e84b35c31ea079f1873d1f","parents":[{"commit":"68f70d59a721d1ad01a20457cd34b803126987c9","subject":"TAO-3993: add call power on/off logic"}],"author":{"name":"Tao.Wang","email":"tao-wt@qq.com","date":"2024-05-28 07:48:33.000000000","tz":480},"committer":{"name":"Tao.Wang","email":"tao-wt@qq.com","date":"2024-05-28 07:48:33.000000000","tz":480},"subject":"TAO-4040: bug fix","message":"TAO-4040: bug fix\\n\\nSigned-off-by: Tao.Wang \\u003ctao-wt@qq.com\\u003e\\nChange-Id: I6b19c4343c2dee6e18ddda72f115f9201ba26090\\n"}\n'
>>> await commit.list_files("59a9264ce673baa992e84b35c31ea079f1873d1f")
'{"/COMMIT_MSG":{"status":"A","lines_inserted":10,"size_delta":354,"size":354},"jenkinsfiles/project_1/auto_test.groovy":{"lines_inserted":7,"size_delta":190,"size":8226}}\n'
>>> await commit.get_content("59a9264ce673baa992e84b35c31ea079f1873d1f", ".flake8")
'[flake8]\nignore =\n    # import at the begining of a module is customary but not required\n    E402\nmax-line-length = 100\n'
>>>
```

### change操作
创建change，修改文件和提交change：
```python
subject = f"test commit message"
change = await project.create_change(subject, "master")
edit = change.edit()
await edit.change_file("path/to/file", "test content")
await edit.delete_file('path/to/file2')
await edit.publish()
await change.submit()
```

### 清理
关闭底层连接：
```python
await gerrit.close()
```

## 代码实现
```python
import io
import base64
import urllib.parse
import ujson
from .async_client import aClient


def strip_magic_prefix(func):
    async def wrapper(*args, **kwargs):
        data = await func(*args, **kwargs)
        return data[5:]
    return wrapper


class aGerrit(aClient):
    def __init__(self, gerrit_url, user, passwd):
        super().__init__(gerrit_url, user, passwd)

    def project(self, project: str):
        return aProject(self, project)

    async def get_change(self, change_id: str):
        change_id = urllib.parse.quote(str(change_id), safe='')
        uri = f'/a/changes/{change_id}'
        resp = await self._request('get', uri)
        return aChange(self, ujson.loads(resp[5:]))


class aProject:
    def __init__(self, gerrit: aGerrit, project: str):
        self._request = gerrit._request
        self.project = project
        proj_id = urllib.parse.quote(project, safe='')
        self.uri_prefix = f"/a/projects/{proj_id}"

    @strip_magic_prefix
    async def list_branches(self) -> str:
        uri = f'{self.uri_prefix}/branches/'
        return await self._request('get', uri)

    @strip_magic_prefix
    async def get_branch(self, branch: str, /) -> str:
        branch_id = urllib.parse.quote(branch, safe='')
        uri = f'{self.uri_prefix}/branches/{branch_id}'
        return await self._request('get', uri)

    async def get_content(self, branch: str, file: str, /) -> str:
        branch = urllib.parse.quote(branch, safe='')
        file = urllib.parse.quote(file, safe='')
        uri = f'{self.uri_prefix}/branches/{branch}/files/{file}/content'
        data = await self._request('get', uri)
        return base64.b64decode(data).decode('utf-8')

    @strip_magic_prefix
    async def create_branch(self, branch: str, revision: str) -> str:
        branch = urllib.parse.quote(branch, safe='')
        data = {"revision": revision}
        uri = f'{self.uri_prefix}/branches/{branch}'
        return await self._request('put', uri, json=data)

    async def create_change(self, subject: str, branch, /, **extra_infos):
        data = {"project": self.project, "subject": subject, "branch": branch}
        data.update(extra_infos)
        resp = await self._request('post', '/a/changes/', json=data)
        return aChange(self, ujson.loads(resp[5:]))

    def commit(self):
        return aCommit(self)


class aCommit:
    def __init__(self, gerrit: aGerrit):
        self._request = gerrit._request
        self.uri_prefix = f"{gerrit.uri_prefix}/commits"

    @strip_magic_prefix
    async def get(self, commit: str):
        commit = urllib.parse.quote(commit, safe='')
        uri = f"{self.uri_prefix}/{commit}"
        return await self._request('get', uri)

    async def get_content(self, commit: str, file: str, /) -> str:
        commit = urllib.parse.quote(commit, safe='')
        file = urllib.parse.quote(file, safe='')
        uri = f"{self.uri_prefix}/{commit}/files/{file}/content"
        data = await self._request('get', uri)
        return base64.b64decode(data).decode('utf-8')

    @strip_magic_prefix
    async def list_files(self, commit: str):
        commit = urllib.parse.quote(commit, safe='')
        uri = f'{self.uri_prefix}/{commit}/files/'
        return await self._request('get', uri)


class aChange:
    def __init__(self, gerrit: aGerrit, change: dict):
        self._request = gerrit._request
        self.info = change
        self.number = change["_number"]
        self.uri_prefix = f"/a/changes/{change['_number']}"

    def edit(self):
        return aEdit(self)

    async def set_to_wip(self):
        uri = f"{self.uri_prefix}/wip"
        data = {"message": "set to work in progress."}
        await self._request('post', uri, json=data)

    async def set_to_ready(self):
        uri = f"{self.uri_prefix}/ready"
        data = {"message": "change is ready for review."}
        await self._request('post', uri, json=data)

    async def abandon(self):
        uri = f"{self.uri_prefix}/abandon"
        data = {"message": "no files changed, so abandon change."}
        await self._request('post', uri, json=data)

    def reviewer(self):
        return aReviewer(self)

    def revision(self, revision="current"):
        return aRevision(self, revision)

    async def submit(self):
        uri = f"{self.uri_prefix}/submit"
        await self._request('post', uri, json={})


class aRevision:
    def __init__(self, change: aChange, revision="current"):
        self.change_number = change.number
        self._request = change._request
        self.uri_prefix = f"{change.uri_prefix}/revisions"
        self.revision = revision

    async def review(self, review_input: dict):
        uri = f'{self.uri_prefix}/{self.revision}/review'
        await self._request('post', uri, json=review_input)


class aReviewer:
    def __init__(self, change: aChange):
        self.change_number = change.number
        self._request = change._request
        self.uri_prefix = f"{change.uri_prefix}/reviewers"

    async def add(self, user: str) -> None:
        data = {"reviewer": user.split('@', 1)[0]}
        await self._request('post', self.uri_prefix, json=data)

    @strip_magic_prefix
    async def list(self,) -> None:
        uri = self.uri_prefix + '/'
        return await self._request('get', uri)


class aEdit:
    def __init__(self, change: aChange):
        self.change_number = change.number
        self._request = change._request
        self.uri_prefix = f"{change.uri_prefix}/edit"

    async def change_file(self, file,
                          content: bytes = b'', fd: io.BufferedReader = None):
        uri = f"{self.uri_prefix}/{file}"
        if content:
            encoded_string = base64.b64encode(content).decode('utf-8')
        elif isinstance(fd, io.BufferedReader):
            encoded_string = base64.b64encode(fd.read()).decode('utf-8')
        data_uri = 'data:text/xml;base64,' + encoded_string
        data = {"binary_content": data_uri}
        codes = {200, 201, 204, 409}
        await self._request('put', uri, expect_codes=codes, json=data)

    async def delete_file(self, file):
        uri = f"{self.uri_prefix}/{file}"
        await self._request('delete', uri)

    async def publish(self) -> bool:
        uri = f"{self.uri_prefix}:publish"
        pub_data = {"notify": "NONE"}
        codes = {200, 201, 204, 409}
        exit_code, _ = await self._request(
            'post', uri,
            exit_code=True,
            expect_codes=codes,
            json=pub_data
        )
        if exit_code != 409:
            return True
        return False

    async def delete(self):
        await self._request('delete', self.uri_prefix)

```

# Confluence模块
目前只实现了获取和更新页面的功能，支持以`Personal Access Token`登录

## 示例
获取页面信息字典，并对其中页面内容和版本信息后，更新到服务端。具体API接口信息请参考[atlassian API文档](https://developer.atlassian.com/server/confluence/expansions-in-the-rest-api/ "Expansions in the REST API")
```python
# 使用Personal Access Token
client = aConfluence(CONFLUENCE_URL, bearer=CONFL_TOKEN)

# 指定返回更多的关于body.storage和version的信息
page = await client.get_page(space, page_id, expand="body.storage,version")

# 更新页面内容和版本信息
page['body']["storage"]['value'] = new_content
page['version']["number"] += 1

# 推送更新到服务端, 并关闭底层连接
await client.update_page(page_id, page)
await client.close()
```

## 代码实现
```python
import ujson
from .async_client import aClient


class aConfluence(aClient):
    def __init__(self, confluence_url, user='', password='', *, bearer=''):
        super().__init__(confluence_url, user, password, bearer=bearer)

    async def get_page(self, space_key, page_id, expand: str = "") -> dict:
        uri = f'/rest/api/content/{page_id}'
        params = {"spaceKey": space_key}
        if expand:
            params.update({"expand": expand})

        return ujson.loads(await self._request('get', uri, params=params))

    async def update_page(self, page_id, data: dict) -> None:
        uri = f'/rest/api/content/{page_id}'
        ujson.loads(await self._request('put', uri, json=data))

```

# Jenkins模块
目前只实现了以下功能：
- 创建Job
- 获取Job信息，判断Job是否存在
- 获取Job配置信息，并更新Job配置

## 示例
首先创建`aJenkins`实例对象：
```python
JENKINS = aJenkins(JENKINS_URL, JENKINS_USER, JENKINS_PASS)
```

获取和更新Job配置：
```python
config = await JENKINS.get_config(job)
await JENKINS.reconfigure(job, config)
```

关闭底层连接：
```python
await JENKINS.close()
```

## 代码实现
```python
from .async_client import aClient


class aJenkins(aClient):
    def __init__(self, jenkins_url, user, password, /):
        super().__init__(jenkins_url, user, password)

    async def get_config(self, job: str) -> str:
        uri = f'/job/{job}/config.xml'
        async with self.client.get(uri) as resp:
            assert resp.status == 200, \
                f"get job '{job}' config failed: {await resp.text()}"
            return await resp.text()

    async def reconfigure(self, job: str, config: bytes) -> None:
        uri = f'/job/{job}/config.xml'
        headers = {'Content-Type': 'text/xml; charset=utf-8'}
        async with self.client.post(uri, headers=headers, data=config) as r:
            assert r.status == 200, \
                f"update job '{job}' config failed: {await r.text()}"

    async def create(self, job: str, config: bytes) -> None:
        uri = '/createItem'
        headers = {'Content-Type': 'application/xml; charset=utf-8'}
        params = {"name": job}

        async with self.client.post(
            uri,
            headers=headers,
            params=params,
            data=config
        ) as resp:
            assert resp.status == 200, \
                f"create job '{job}' failed: {await resp.text()}"

    async def is_exists(self, job: str) -> bool:
        try:
            await self.get_info(job)
        except Exception:
            return False
        else:
            return True

    async def get_info(self, job: str) -> str:
        uri = f'/job/{job}/api/json'
        async with self.client.get(uri) as resp:
            assert resp.status == 200, \
                f"get job '{job}' info failed: {await resp.text()}"
            return await resp.text()

```

# 基础模块
`async_client`模块中主要是用`aiohttp`模块共享的异步的`aiohttp.ClientSession`对象，并对异常做了统一的结构化处理。

## 代码实现
```python
import aiohttp


class aClient:
    def __init__(self, url, user='', passwd='', *, bearer=''):
        if user and passwd:
            auth = aiohttp.BasicAuth(user, password=passwd)
            self.client = aiohttp.ClientSession(url, auth=auth)
        elif bearer:
            headers = {"Authorization": f"Bearer {bearer}"}
            self.client = aiohttp.ClientSession(url, headers=headers)
        else:
            self.client = aiohttp.ClientSession(url)

    async def _request(self, method, uri, /, *,
                       exit_code=False, expect_codes: set = {200, 201, 204},
                       **requests_params):
        method_obj = getattr(self.client, method)
        async with method_obj(uri, **requests_params) as resp:
            assert resp.status in expect_codes, \
                f"{method} '{uri}' failed: {await resp.text()}"
            code = resp.status
            data = await resp.text()
        if exit_code:
            return code, data
        else:
            return data

    async def close(self):
        await self.client.close()

```
