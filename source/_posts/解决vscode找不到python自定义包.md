---
title: 解决vscode找不到python自定义包
date: 2022-05-18 22:00:21
tags: python
---

# 解决vscode找不到python自定义包

+ 如果提示找不到当前项目的xxx module,可以将当前项目或者xxx的路径加入到pythonpath环境变量中

+ 在.vscode的settings.json中, 添加以下内容, 解决terminal运行python程序时找不到自定义模块的问题

```json
"python.autoComplete.extraPaths": ["${workspaceFolder}", "/path/to/your/module"],
"terminal.integrated.env.windows": {
        "PYTHONPATH": "${workspaceFolder};${env:PYTHONPATH};"
}
```

+ 在.vscode的launch.json中, 添加以下内容, 解决调试模式找不到自定义模块的问题

```json
"env": {"PYTHONPATH":"${workspaceFolder};${env:PYTHONPATH}"}
```

## 说明

 `python.autoComplete.extraPaths`指定vscode的python插件寻找其他包的位置, `terminal.integrated.env.windows`指定terminal的pythonpath环境变量, pycharm不会报找不到自定义包的原因是pycharm会自动把项目路径添加到pythonpath环境变量中, vscode则需要手动添加

