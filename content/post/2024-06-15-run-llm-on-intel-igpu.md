---
layout: post
title:       "Run LLM on Intel Iris Xe GPU Using IEPX-LLM + Ollama"
subtitle:    ""
description: "For windows demostration only."
date:        "2024-06-15"
author:     "Jamie Zhang"
image:       "/img/ai-01.jpg"
tags:        ["LLM", "iGPU"]
categories:  ["AI" ]
---

In the realm of AI and machine learning, optimizing performance often hinges on utilizing GPU acceleration. However, Intel GPUs traditionally have not supported running AI workloads directly with popular frameworks like TensorFlow or PyTorch [5]. To address this, developers can turn to IEPX-LLM, a specialized library tailored for Intel's XPU architecture.




# Steps to Run LLM on Intel GPU with IEPX-LLM + Ollama Locally
## Install Prerequisites
### (Optional) Update GPU Driver
> Tips
> :information_source:
> It is recommended to update your GPU driver, if you have driver version lower than 31.0.101.5122. Refer to here for more information.

Download and install the latest GPU driver from the official Intel download page. A system reboot is necessary to apply the changes after the installation is complete.

> The process could take around 10 minutes. After reboot, check for the Intel Arc Control application to verify the driver has been installed correctly. If the installation was successful, you should see the Arc Control interface similar to the figure below
<img src='/img/2024-06-15-run-llm-on-intel-igpu/gpu-driver-version.jpg' style="height: 340px;margin-left: 0px;"/>

### Setup Python Environment
Visit [Miniforge installation page](https://conda-forge.org/download/), download the Miniforge installer for Windows, and follow the instructions to complete the installation.
<img src='/img/2024-06-15-run-llm-on-intel-igpu/miniforge-01.jpg' style="height: 340px;margin-left: 0px;"/>

After installation, open the Miniforge Prompt, create a new python environment llm:


```
conda create -n llm python=3.11 libuv
```
Activate the newly created environment llm:

```
conda activate llm
```


## Install ipex-llm
With the llm environment active, use pip to install ipex-llm for GPU. Choose either US or CN website for extra-index-url:

    * US
        pip install --pre --upgrade ipex-llm[xpu] --extra-index-url https://pytorch-extension.intel.com/release-whl/stable/xpu/us/

    * CN
        pip install --pre --upgrade ipex-llm[xpu] --extra-index-url https://pytorch-extension.intel.com/release-whl/stable/xpu/cn/
> If you encounter network issues while installing IPEX, refer to this [guide](https://ipex-llm.readthedocs.io/en/latest/doc/LLM/Overview/install_gpu.html#install-ipex-llm-from-wheel) for troubleshooting advice.


## Verify Installation
- Open the Miniforge Prompt and activate the Python environment llm you previously created:
Step 1: Runtime Configurations  
```
conda activate llm
```
- Set the following environment variables according to your device:
>
    * Intel iGPU
        set SYCL_CACHE_PERSISTENT=1  
        set BIGDL_LLM_XMX_DISABLED=1  

    * Intel Arc A770
        set SYCL_CACHE_PERSISTENT=1
     
Step 2: Run Python Code
- Launch the Python interactive shell by typing python in the Miniforge Prompt window and then press Enter.

- Copy following code to Miniforge Prompt line by line and press Enter after copying each line.
```
import torch 
from ipex_llm.transformers import AutoModel,AutoModelForCausalLM    
tensor_1 = torch.randn(1, 1, 40, 128).to('xpu') 
tensor_2 = torch.randn(1, 1, 128, 40).to('xpu') 
print(torch.matmul(tensor_1, tensor_2).size()) 
```
It will output following content at the end:
> torch.Size([1, 1, 40, 40])

- To exit the Python interactive shell, simply press Ctrl+Z then press Enter (or input exit() then press Enter).

> PS:
> If hit error on import the iepx, that might be caused by configure issue, or you may consider to reinstall the iepx-llm.


## Run Llama3 using Ollama
### Initialize Ollama
Activate the llm conda environment and initialize Ollama by executing the commands below. A symbolic link to ollama will appear in your current directory.
```
# Please run the following command with administrator privilege in Miniforge Prompt.
conda activate llm
init-ollama.bat
```
In this step, it will create two soft linkages, this step is very important, the ollama command will be use later to run LLM model, rather than the self-manual installed ollama tool.
<img src='/img/2024-06-15-run-llm-on-intel-igpu/ollama-init.jpg' style="height: 140px;margin-left: 0px;"/>

> If you have installed higher version ipex-llm[cpp] and want to upgrade your ollama binary file, don’t forget to remove old binary files first and initialize again with init-ollama or init-ollama.bat.

**Now you can use this executable file by standard ollama’s usage.**

### Run Ollama Serve
Please run the following command in Miniforge Prompt.
```
set no_proxy=localhost,127.0.0.1
set ZES_ENABLE_SYSMAN=1
set OLLAMA_NUM_GPU=999

ollama serve
```
> 1. To allow the service to accept connections from all IP addresses, use OLLAMA_HOST=0.0.0.0 ./ollama serve instead of just ./ollama serve.
> 2. You can add this ollama command to PATH for later use purpose.

Keep the Ollama service on and open another terminal and run llama3 with ollama run:
```
set no_proxy=localhost,127.0.0.1
ollama run llama3:8b-instruct-q4_K_M
```
> Notes:
> Here we just take llama3:8b-instruct-q4_K_M for example, you can replace it with any other Llama3 model you want.

Below is a sample output on Intel Iris Xe GPU.
<img src='/img/2024-06-15-run-llm-on-intel-igpu/llm-running-01.png' style="height: 680px;margin-left: 0px;"/>

# Additional Resources
[IPEX-LLM Documentation](https://ipex-llm.readthedocs.io/)  
[Run Llama 3 on Intel GPU using ollama with IPEX-LLM](https://ipex-llm.readthedocs.io/en/latest/doc/LLM/Quickstart/llama3_llamacpp_ollama_quickstart.html)  
[Intel® Extension for PyTorch* for GPUs](https://www.intel.com/content/www/us/en/developer/articles/technical/introducing-intel-extension-for-pytorch-for-gpus.html)