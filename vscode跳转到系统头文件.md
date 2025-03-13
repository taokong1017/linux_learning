## vscode跳转到系统
&emsp;&emsp;vscode允许通过remote-ssh插件，连接到远程服务器。偶尔你可能会遇到，在vscode中，总是跳转到系统头文件而头疼。

&emsp;&emsp;这个问题是可以解决的，其解决办法就是，在.vscode目录下，添加c_cpp_properties.json文件，具体内容如下：
```c
{
    "configurations": [
        {
            "name": "Linux",
            "includePath": [
                "${workspaceFolder}/**" //包含当前工作区下的所有文件
            ],
            "defines": [],
            "compilerPath": "",
            "cStandard": "c89",
            "cppStandard": "c++17",
            "browse": {
                "path": [
                    "${workspaceFolder}"
                ],
                "limitSymbolsToIncludedHeaders": true,
                "databaseFilename": ""
            }
        }
    ],
    "version": 4
}
```
