![image](https://github.com/user-attachments/assets/cddcf63f-acad-4976-99b6-eb5b72f60ecf)# Efficient-Hardware-Software-Codesign-for-accelerating-Vision-Transformer-ViT-Inference



![image](https://github.com/user-attachments/assets/701de01b-61c4-4f5f-9076-6cb38da17f52)

## Introduction

As the presence of edge IoT devices increases, they become more vulnerable to malware attacks. While many malware attack detection strategies have been proposed over the years, Vision Transformers (ViTs) are increasingly being used for malware detection. Given ViTs' robust self-attention mechanism, they achieve good accuracy when they are compressed and deployed on edge devices.

Typically, given the parallelism in design, transformers and ViTs are executed on graphic processing units (GPUs). However, most applications running on edge devices do not require the high parallel processing power of GPUs. Also, the usage of GPUs is not always possible given their high area and power consumption. 

Using a similar argument, ViTs should not be implemented entirely on hardware (high area and power consumption). Since the ViTs are designed only for malware detection, they cannot be used by other applications. Implementation of ViT on the processor, with its serial execution results in slow inference. Even if multithreading is enabled, it cannot rival the parallel processing by GPU or accelerators.

With all these factors in mind, we aim to accelerate ViT inference on edge devices while trying to minimize the hardware overhead. In this work, we will consider multiple possibilities of co-design and provide a tradeoff between the inference time and hardware overhead. At the end of the proposed work (in Stage 2), we aim to provide the designer with the state space exploration of all possible hardware-software combinations.

## Background Research

### Vision Transformers

The following figure shows the block diagram of the proposed ViT.

![image](https://github.com/user-attachments/assets/7fb31df5-e926-4882-b48a-6d64e460ca0e)


**Figure 1 - ViT Block Diagram**

The function of each block in the ViT is given below:
- **Patch Encoder**: The given image is divided into 256 patches, which are encoded and passed to the encoder.
- **Layer Normalisation**: This is used to stabilize the transformer during the training phase and allows easy convergence. It arranges all the weights around the mean value. In the inference, the weights learned during the training are used to ensure stability of the network.
- **Dense Layer**: The dense layers are fully connected layers that allow interaction between appended heads and provide the final classification. In our implementation, the first and second dense layers have outputs of 128, while the third dense layer has an output of 2 (the number of classes).
- **Multi-head Attention**: It is the heart of the ViT. Inside it, each input vector is broken into a query (Q), key (K), and value (V) vector. The attention is given by:

To allow parallel processing, the inputs are divided into “heads” (4 in our case). Hence, at a given time in our implementation, each attention module is responsible for 8✕8 array multiplication.

### VEGA AS1061

VEGA AS1061 is a six-stage, 64-bit, in-order, and RV64IMAFDC ISA compatible processor. To connect with external peripherals, the processor uses AHB/AXI interface. The processor is designed for high-performance embedded, consumer electronics, motor control, and industrial automation applications.

**Figure 2: VEGA AS1061 Block Diagram**

## Goal and Objectives

### Goal
To optimize the inference of the Vision Transformer using efficient hardware-software co-design.

### Objectives
- While improving the inference, minimum hardware overhead should be added.
- The proposed ViT IP should use the AXI interface for connection with the VEGA AS1061 processor.
- We aim to efficiently parallelize the multi-head attention module, as this particular module is repeatedly used in the ViT.

## Design Process

### Problem Statement

As shown in the attention equation, the multihead attention scheme consists of two matrix multiplications. In the first part, Q and K matrices are multiplied. In the second part, the result obtained by the previous part is multiplied with the V matrix. It is to be noted that the result of the first part is used in the second part, and hence the two multiplications cannot be made parallel. However, with the clever usage of hardware-software co-design, they can be pipelined.

The operations per head can be broken into 4 phases:
1. Fetching Q and K
2. Matrix multiplication of Q and K 
3. Division and softmax function 
4. Matrix multiplication of previous stage with V

While each head can be implemented in parallel, designing 8 64✕64 matrix multiplications results in additional hardware. Also, given that Q and K are large matrices, there is a significant delay in bringing the data from DRAM to the processor. With these factors in mind, we implement only one head in the hardware. While the head implements on hardware, we fetch data from the memory, hiding the latency incurred to fetch the data.

To ensure the implementation of softmax and division on hardware, we use the circuit proposed in [1] and [2]. For the divider, we use the error-tolerable nature of the softmax function to accelerate the division.

### Functional Specification

#### Proposed Design

The proposed data pipeline is shown in the table below.

| Pipeline Stage       | Events happening                                                                 |
|----------------------|----------------------------------------------------------------------------------|
| Fetch                | Matrices Q and K are fetched from DRAM/BRAM to initiate execution                |
| Attention Phase 1    | 1. Matrix Multiplication of Q and K vectors <br> 2. Processor initiates the fetching of matrix V |
| Softmax and Division | 1. Division by the usage of approximate divider <br> 2. Softmax implementation in the hardware module <br> 3. Processor completes fetching of matrix V |
| Attention Phase 2    | Matrix multiplication of the result obtained in the previous stage with matrix V |

**Table 1: Proposed Pipelined Implementation**

## Analysis of Final Design

We aim to test the design by putting the proposed transformer on the Programmable Logic portion of the provided SoC, using DMA for data acquisition from DRAM and the VEGA AS1061 processor as the Programming System. To compare the results, we will use `XTime_SetTime` provided by Xilinx to compare the execution time of PS and our proposed logic.


## Results and Discussion

### Transformer Modelling Process

![image](https://github.com/user-attachments/assets/8bfa525b-936b-415a-b116-733a65035f17)
![image](https://github.com/user-attachments/assets/9b1de4e6-667f-4f76-9c90-ba9cf436b871)
![image](https://github.com/user-attachments/assets/028a2db4-103c-4b5b-bda0-1540a0af6094)


**Figure 7**: ViT Inference Accuracy

### Transformer IP Design

![image](https://github.com/user-attachments/assets/860e9423-a606-4980-a5cc-929b9ab1fde5)


IP interface with MicroBlaze

In this design, we design a basic ViT accelerator (not the one utilized in the problem statement, but a simple one). The designed IP has two main parts:
1. Accelerator control
2. Spad_Abiter

In the accelerator used for this purpose, systolic arrays have been used to implement the transformer. The main aim of the accelerator design was to showcase the IP design using AXI interface. Note that the entire design uses the Programmable Logic portion of the Xilinx Zedboard, since MicroBlaze is a soft processor. We perform synthesis and implementation using Xilinx Vivado 2019.1. The power usage and resource utilization are shown below.

