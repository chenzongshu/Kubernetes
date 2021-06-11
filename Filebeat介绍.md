

# 配置

## log input

### paths

我们可以指定一系列的 paths 作为信息输入源，在指定 path 的时候，注意以下规则：

1. 指定的路径必须是文件，不能是目录。
2. 支持 Glob 模式。
3. 默认支持递归路径，如 `/**/` 形式，Filebeat 将会展开 8 层嵌套目录。

假如指定了 `/home/data/**/my*.log`, filebeat 将会把 `/**/` 翻译成 8 层的子目录，如下：

```
/home/data/my*.log
/home/data/*/my*.log
/home/data/*/*/my*.log
/home/data/*/*/*/my*.log
/home/data/*/*/*/*/my*.log
/home/data/*/*/*/*/*/my*.log
/home/data/*/*/*/*/*/*/my*.log
/home/data/*/*/*/*/*/*/*/my*.log
/home/data/*/*/*/*/*/*/*/*/my*.log
```

### exclude_lines

可指定多个正则表达式，来去除某些不需要上报的行。例如：

```yaml
filebeat.inputs:
- type: log
  ...
  exclude_lines: ['^DBG']
```

该配置将会去除以 `DBG` 开头的行。

### include_lines

可指定多项正则表达式，来仅上报匹配的行。例如：

```yaml
filebeat.inputs:
- type: log
  ...
  include_lines: ['^ERR', '^WARN']
```

该配置将会仅上报以 `ERR` 和 `WARN` 开头的行。

如果同时指定了 exclude_lines 和 include_lines ，Filebeat 将会先校验 include_lines，再校验 exclude_lines

### exclude_files

```yaml
exclude_files: ['.gz$']
```

> 这里需要注意的是，不管是 exclude_files，还是 exclude_lines、include_lines, 声明正则的时候，最好使用单引号引用正则表达式，不要用双引号。否则 yaml 会报转义问题

### harvester_buffer_size

读文件时的 buffer 大小

### max_bytes

表示一条 log 消息的最大 bytes 数目。超过这个大小，剩余就会被截断。
默认值为 10485760(即 10MB)。

### ignore_older

ignore_older 表示对于最近修改时间距离当前时间已经超过某个时长的文件，就暂时不进行处理。默认值为 0，表示禁用该功能。

注意：ignore_older 只是暂时不处理该文件，并不会在 Registrar 中改变该文件的状态。

