---
layout: post
title:  "Optimizing Sparse Matrix-Vector Multiplication on Emerging Many-Core Architectures, 2018"
date:   2019-06-27 18:43:59
author: Sangki Park
categories: Paper_Review
tags:	
cover:  "/assets/resize_instacode.jpg"
use_math: true
---

이번 포스팅에서 다룰 논문은 2018년도 HPCC에 실린 ["Optimizing Sparse Matrix-Vector Multiplication on Emerging Many-Core Architectures"][paper_arxiv]이다.  
이 논문은 Intel Knights Landing (KNL) XeonPhi 와 ARM기반 FT-2000Plus (FTP) 아키텍쳐에서 Sparse 행렬 표현방식을 바꿨을때 sparse matrix vector multiplication (SpMV)에 어떤 영향이 있는지에 대한 실험을 진행하였다.  
956개의 데이터셋 종류를 사용했으며, 5가지의 SpMV 표현방식을 (COO, CSR, CSR5, ELL, SELL, HYB) 이용했다. 머신러닝을 통해 아키텍쳐와 프로그램 input에 따른 적합한 sparse 행렬 표현을 선택하도록 하는 예측모델을 만들었으며, 해당 모델은 KNL, FTP에서 각각 95%, 91%의 "best available performance"를 달성하였고 profiling overhead가 없었다고 한다.

해당 논문에서 밝힌 핵심 contributions는 다음과 같다.
>- It presents an extensible framework to evaluate SpMV performance on KNL and FTP, two emerging many-core architectures for HPC;
>- It is the first comprehensive study on the impact of sparse matrix representations on KNL and FTP;
>- It develops a novel machine learning based approach choose the right representation for a given architecture, delivering significantly good performance across manycore architectures;
>- Our work is immediately deployable as it requires no modification to the program source code.

---
SpMV는 보통 $y = Ax$ ($A\in \mathbb R^{M\times N}, x\in \mathbb R^{N}, y\in \mathbb R^{M}$) 으로 표현하게 된다. 이때 행렬 $A$는 Fig.1 처럼 sparse한 행렬이라고 가정한다. 이런 희소행렬 $A$를 표현할때는 여러가지 방식이 있는데, 해당 논문에서 언급하고 있는 방식은 6가지 이다. (Table 1 참고)
1. COO (Coordinate 또는 IJV format)
    - 비교적 간단. $row$, $col$ 벡터에는 데이터가 존재하는 index를 저장하고 $data$ 벡터에는 non-zero value 만 저장
1. CSR (Compressed Sparse Row)
    - 가장 많이 쓰임. $ptr$은 non-zero 값의 갯수를 row index별로 차례로 저장
    - $col$은 데이터가 있는 col index를 row-major 로 저장
    - $data$는 실제 데이터 저장
1. CSR5
    - Table 1의 예시는 $\mathbb w=\sigma=2$ 일때
1. ELL (ELLPACK)
    - vector architecture 에 적합. Table 1 예시는 $K=3$ 일때
1. SELL & SELL-C-$\sigma$ (Sliced ELL)
    - ELL의 확장. Table 1 예시는 $C=2, \sigma=4$ 일때
1. HYB (ELL과 COO를 섞음)
    - Table 1 예시는 $K=2$ 일때


![fig1](/assets/img/2019-6-27-OptimiSpMV/fig1.PNG){: width="50%" height="50%"}
![fig1](/assets/img/2019-6-27-OptimiSpMV/fig1.PNG)
![fig1](/assets/img/2019-6-27-OptimiSpMV/fig1.PNG){: .img_sameline}
<center><img src="https://psk9211.github.io/assets/img/2019-6-27-OptimiSpMV/table1.PNG" width="70%"></center>
<center><img src="/assets/img/2019-6-27-OptimiSpMV/table1.PNG" width="40%"></center>
<img src="/assets/img/2019-6-27-OptimiSpMV/table1.PNG" width="500">
<img src="/assets/img/2019-6-27-OptimiSpMV/table1.PNG" width="500px">

---
사용한 하드웨어는 **FT-2000Plus** (FTP)와 Intel KNL 프로세서이다.

|   |FTP|Intel KNL|
|---|:--- |:---|
|Core|ARMv8 기반 64-core|72-core|
|Freq.|2.4GHz|1.3GHz|
|Peak-performance|512GFLOPS|6TFLOPS (single-precision)<br>3TFLOPS (double-precision)|
|OS|Linux with v4.4.0 kernel|Linux with v3.10.0 kernel|
|Compiler|gcc v6.4.0 (-O3)|icc v17.0.4 (-O3)|
|Dataset|SuiteSparse matrix collection|SuiteSparse matrix collection|


<center><img src="https://psk9211.github.io/assets/img/2019-6-27-OptimiSpMV/fig3.PNG" width="50%"><img src="https://psk9211.github.io/assets/img/2019-6-27-OptimiSpMV/fig2.PNG" width="50%"></center>
<center><img src="https://psk9211.github.io/assets/img/2019-6-27-OptimiSpMV/al1.PNG" width="50%"></center>

실제 코드 구현은 위의 알고리즘1 처럼 진행 되었다. OpenMP를 사용하여 thread 단위 병렬화를 하였으며, 구현 알고리즘은 각 방식마다 다르게 적용된다.

---

SpMV 성능은 여러 요인에 의해 영향을 받는데, memory allocation, code optimization strat, sparse matrix representation 등이 있다. 논문에서는 희소행렬 표현 방식에 중점을 두기 위해 나머지 두 조건을 "best" 상태에서 진행하려 하였다. 따라서 논문은 Non-uniform memory access (NUMA)와 code vectorization이 SpMV에 미치는 영향을 조사하였다.

NUMA-aware 방식을 이용해 FTP 프로세서에서 최적화를 진행하였을 때 CSR, CSR5, **ELL**, SELL, HYB 방식에서 각각 1.5x, 1.9x, **6.0x**, 2.0x, 1.9x 배 만큼의 속도 향상이 있었다고 한다. (Fig.4)  
KNL에서 vectorization의 결과는 Fig.5 와 같다.
<center><img src="https://psk9211.github.io/assets/img/2019-6-27-OptimiSpMV/fig4.PNG" width="40%"><img src="https://psk9211.github.io/assets/img/2019-6-27-OptimiSpMV/fig5.PNG" width="50%"></center>
<br>
Fig.6의 경우 FTP가 KNL에 비해 얼마나 더 빠른가를 보여주고 있다. CSR, CSR5, ELL, SELL, HYB 방식에서 각각 1.9x, 2.3x, 1.3x, 1.5x, 1.4x 배 만큼 차이가 난다고 한다. 논문에서는 이 차이의 이유를 속도가 빠른 MCDRAM 때문이라고 하고 있다.
<br><br>
<center><img src="https://psk9211.github.io/assets/img/2019-6-27-OptimiSpMV/fig6.PNG" width="70%"></center><br>

FTP와 KNL을 각각 64, 272 thread로 non-zero값의 개수가 다른 행렬들을 다른 방식으로 돌린 결과는 Fig.7과 Table.3 과 같다.<br><br>
<center><img src="https://psk9211.github.io/assets/img/2019-6-27-OptimiSpMV/fig7.PNG" width="70%"></center><br>
<center><img src="https://psk9211.github.io/assets/img/2019-6-27-OptimiSpMV/table3.PNG" width="70%"></center><br>

Table.3은 특이하게 단일 방식을 사용했을때와 각각 optimal 한 경우를 사용했을때 느려진 정도를 나타내고 있다. FTP에서는 HYB와 SELL 방식이, KNL에서는 CSR 방식이 가장 *덜 느려졌*으므로 단일 방식을 사용한다면 이걸 사용하는게 좋다고 할 수 있다.

---

이제 위 실험 결과를 통해 predictor를 만들 차례이다. Fig.9 처럼 모델을 디자인하고, feature로는 Table.4 의 metric 들을 사용하였다. 전체 데이터 중 80%를 train에, 나머지 20%를 test에 사용하였다. 모델은 feature 값과 optimal representation label 간 correlation을 찾는 방식으로 설계되었며, 결과물은 decision-tree based model 이다.  
훈련 데이터를 두 플랫폼에 대해 수집하는데 3일이 걸렸으며, 실제 모델 building에는 10ms 밖에 안걸렸다고 한다. 또한 수집한 데이터는 $0$ ~ $1$ 사이의 값으로 scaling 되었다.  
실험 결과는 Fig.10 과 같다. 

<br>
<center><img src="https://psk9211.github.io/assets/img/2019-6-27-OptimiSpMV/fig9.PNG" width="50%"><img src="https://psk9211.github.io/assets/img/2019-6-27-OptimiSpMV/table4.PNG" width="50%"></center><br>
<center><img src="https://psk9211.github.io/assets/img/2019-6-27-OptimiSpMV/fig10.PNG" width="80%"></center><br>

논문에서 사용한 decision-tree 방식 (DTC) 이외의 다른 방식들의 예측 정확도는 Fig.11에 나타나 있다. FTP에 대해서는 DTC 방식이 가장 좋으며, KNL에서는 VC, KNC, LR, RFC, DTC가 모두 93% 이상의 정확도 (내가 보기에) 를 보여주고 있다.
<center><img src="https://psk9211.github.io/assets/img/2019-6-27-OptimiSpMV/fig11.PNG" width="70%"></center><br>

---

논문에 대해 3줄 요약하자면
1. NUMA binding과 vectorization, SpMV 저장 방식이 SpMV 성능에 미치는 영향을 조사하고,
1. optimal한 SpMV 저장방식은 연산장치 아키텍쳐와 input의 sparsity 패턴에 따라 다름을 밝혔으며,
1. sparse matrix의 여러 metric들을 이용해 가장 적합한 sparas matrix representation을 찾는 머신러닝 모델 (decision-tree based)을 만들었다.

로 정리해 볼 수 있겠다.


*아직 결과 분석이 충분치 않으므로 추후 수정을 할 예정



[paper_arxiv]:  https://arxiv.org/abs/1805.11938