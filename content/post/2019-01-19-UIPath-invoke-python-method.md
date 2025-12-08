---
title: "How to invoke python scripts in UIPath"
subtitle: ""
excerpt: ""
description: "simple demostration about how to implement python script invocation in UIPath"
date: 2019-01-19 12:10:27
author: "Jamie Zhang"
image: "/img/background-03.jpg"
tags:
    - Python
    - UIPath
URL: "/2019/01/19/UIPath-invoke-python-method/"
categories: [ Others ]
---
UIPath是一个非常不错的RPA(Robotic Process Automation) 工具和平台，并且有开源社区版本可以方便RPA爱好者去尝试，本文是介绍如何在UIPath下调用python的脚本中的方法并提取返回值

# 模拟问题场景
uipath加载python脚本，并根据传入的配置文件，返回配置文件中的符合条件的配置信息

# 解决步骤  
## 预置条件：
a. python脚本pmt_detection.py， 内容如下：
```
from configparser import ConfigParser
def getPmtReqsStatToBeSync(configFilePath):
	"""
	read config item in rpa-config.ini
	:param: configFilePath - config file
	:return: PMT.sync_scan_on_status
	"""
	cfg = ConfigParser()
	cfg.read(configFilePath)

	return cfg.get("PMT", "sync_scan_on_status").split(',')
```
b. 配置文件rpa-config.ini,内容如下：
```
[PMT]
sync_scan_on_status=SUBMITTED,INPROGRESS
rm_scan_on_status=CANCELLED,COMPLETED

[XXX]
doc_type_list=ID,PASSPORT
```
我们本次的目标是，用getPmtReqsStatToBeSync去拿到配置文件中的key sync_scan_on_status对应的value，即SUBMITTED,INPROGRESS  
c.uipath项目已创建且python activities已安装

## 设置PYTHON_HOME并用变量保存
避免每次调用python脚本的时候都要hardcode python env home 地址，及python.exe所在的目录  
a. 我们现在系统环境变量中设置PYTHON_HOME,如下图
![](/img/2019-01-19-UIPath-invoke-python-method/15532408-48a93520172f1317.png)
b. 在UIpath activities中找到'Get Environment Variable'并拖入使用，配置如下图，用变量保存下python home的地址
![](/img/2019-01-19-UIPath-invoke-python-method/15532408-b9231e50939f7b28.png)

## 使用Python activities下面的组件来调用python脚本
a. 添加python scope及配置
![](/img/2019-01-19-UIPath-invoke-python-method/15532408-b3d0136be36e7810.png)
截了其它例子中的图(请姑且认为py_home跟python_home是同一个变量)
需要补充说明的是,除了Path需要配置之外，Target和Version也请选择正确否则会得到无法加载script的错误且没有具体的错误日志  
b. 在python scope的Do区域添加Load Python Script并设置脚本路径和result - 变量保存python object
![](/img/2019-01-19-UIPath-invoke-python-method/15532408-91e3fecb9fd03402.png)  
c. 在添加invoke python method组件，配置如下
![](/img/2019-01-19-UIPath-invoke-python-method/15532408-aa7b7078eba0b3e7.png)
Instance即 在上一步加载进来的脚本对象，Name即方法名，本例中方法名为：getPmtReqsStatToBeSync，输出同样是pythonObject
> 在我们的需求场景中，需要传递一个配置文件地址，即使它与脚本放在同一个目录下(e.g 项目的根目录，与Main.xaml)，uipath-python也无法加载到（os.path.dirname(os.path.abspath(__file__)) or os.getcwd(), both failed）
所以我们假设rpa-config.ini 和pmt_detection.py放在项目的根目录下，首先需要获取项目的当前path，所以最后Input parameters的配置为：{Environment.CurrentDirectory + “/rpa-config.ini”}
![](/img/2019-01-19-UIPath-invoke-python-method/15532408-8f34731c7c4b4884.png)

d. 拿到了方法的执行之后的返回对象之后，我们还需要添加get python object来获取到返回值，我们的例子中返回值是个tuple即数组或者ArrayList。
![](/img/2019-01-19-UIPath-invoke-python-method/15532408-f29b169d4d7b725d.png)
Input 为‘invoke python method’的返回对象，TypeArgument是要转化的对象类型，本例为：System.String[],输出到变量中。
至此，调用python 方法完成，返回结果变量可以在后面的flow正常使用了

打印日志可以查看输出结果：

    "[" +correlationid +"] Payment sync config loaded, payment sync status are: " +"".Join(",",pmt_sync_status) 
![](/img/2019-01-19-UIPath-invoke-python-method/15532408-d07d7cb65ed442b0.png)

*备注*
欢迎多多指正
