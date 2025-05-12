## vscode跳转到系统
&emsp;&emsp;vscode允许通过remote-ssh插件，连接到远程服务器。偶尔你可能会遇到，在vscode中，总是跳转到系统头文件而头疼。

&emsp;&emsp;这个问题是可以解决的，其解决办法就是，在.vscode目录下，添加c_cpp_properties.json文件，具体内容如下：
```c
{
    "configurations": [
        {
            "name": "Linux",
            "includePath": [
                "${workspaceFolder}/**"
            ],
            "defines": [],
            "compilerPath": "/usr/bin/gcc",
            "cStandard": "c17",
            "cppStandard": "c++17",
            "intelliSenseMode": "linux-gcc-x64",
            "compilerArgs": [
                "-nostdinc",
                "-I${workspaceFolder}/include",
                "-v"
            ],
            "browse": {
                "limitSymbolsToIncludedHeaders": true
            }
        }
    ],
    "version": 4
}
```
