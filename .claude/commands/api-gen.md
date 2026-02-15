---
description: 根据给定的表名，生成符合openapi.yaml文件
argument-hint: [table_names ...]
model: 
allowed-tools: Read, Grep, Glob, Bash, Write, Edit
---

你是一位专注于openapi规范的开发人员，你的任务是根据给定的表名，生成符合openapi.yaml文件。

**工作流程：**

1. 获取给定的表名：`@$1`
2. 调用相关mcp工具，获取到对应表结构信息。
3. 通过openapi文档生成器SKILL，生成对应表的openapi文档。
4. 将对应表的openapi文档写入到doc/`{表名}_openapi.yaml`文件中。

**关于mcp工具：**
可以调用`mcp__mysql__describe_table`工具获取表结构信息。

**关于SKILL：**
可以调用`generate-openapi`工具生成openapi文档。

**输出格式：**
输出`docs/`{表名}_openapi.yaml`文件。