---
title: PyTorch CUDA 드라이버 버전 불일치 해결법
author: jisu
date: 2025-03-04 11:13:00 +0900
categories: [Deep Learning]
tags: [PyTorch]
pin: false
math: true
mermaid: true
comments: true
---

## AWS EC2환경에서 PyTorch 버전
내가 프로젝트 시 쓰던 가상환경 Project에 PyTorch가 설치되어있었고 모델을 돌리는데 

```python
torch.cuda.is_available()
```

이 코드가 `False`로 떴었다.

AWS EC2 인스턴스를 확인해보니 `g4dn.xlarge`로 GPU를 사용할 수 있는 버전이였다.

일단, NVIDIA 드라이버와 CUDA가 정상적으로 설치되었는지 확인해봤다.

```bash
nvidia-smi
```

```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 510.47.03    Driver Version: 510.47.03    CUDA Version: 11.6     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Tesla T4            On   | 00000000:00:1E.0 Off |                    0 |
| N/A   28C    P8    13W /  70W |      0MiB / 15360MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

```bash
nvcc --version
```

```
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2022 NVIDIA Corporation
Built on Thu_Feb_10_18:23:41_PST_2022
Cuda compilation tools, release 11.6, V11.6.112
Build cuda_11.6.r11.6/compiler.30978841_0

```

확인해보니 CUDA 버전은 11.6이고 PyTorch 버전은 2.4.1 이였다.

```bash
pip show torch
```

```
Name: torch
Version: 2.4.1
Summary: Tensors and Dynamic neural networks in Python with strong GPU acceleration
Home-page: https://pytorch.org/
Author: PyTorch Team
Author-email: packages@pytorch.org
License: BSD-3
```

PyTorch 공식 사이트에서 확인해보니 PyTorch와 CUDA가 버전이 호환되지 않아서 발생하는 문제였다. 
> 참고: [https://pytorch.org](https://pytorch.org)

그래서 다시 PyTorch를 uninstall하고 CUDA와 호환되는 버전으로 재설치하였다.

```bash
pip uninstall torch
pip install torch --index-url https://download.pytorch.org/whl/cu116
```

다시 설치하고 나서 확인해보니 성공적으로 GPU를 사용하여 모델을 돌릴 수 있게 되었다.

```python
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
device
```

```
device(type='cuda')
```





