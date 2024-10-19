## 无网路现象
&emsp;&emsp;Ubuntu操作系统，在强制进行虚拟机关机后，再启动虚拟机，发现Ubuntu网路无法连接，同时网路图标也消失了。

## 解决办法
&emsp;&emsp;下面是具体的解决办法，其包含5个步骤：

***步骤1：***
```c
sudo service network-manager stop
```
&emsp;&emsp;停止网路服务。

***步骤2：***
```c
sudo rm /var/lib/NetworkManager/NetworkManager.state
```
&emsp;&emsp;删除网路状态临时文件。

***步骤3：***
```c
sudo service network-manager start
```
&emsp;&emsp;启动网路服务。

***步骤4：***
```c
sudo vim /etc/NetworkManager/NetworkManager.conf
```
&emsp;&emsp;将标签【ifupdown】中managed属性改为true。

***步骤5：***
```c
sudo service network-manager restart
```
&emsp;&emsp;重启网路服务，至此，便可以重新看到Ubuntu网络图标，同时网络服务正常。

