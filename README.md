# Realizing full DSP data processing chain in IWR6843AOP

In this repository, I will introduce how to realize full DSP DPC for out of box demo in IWR6843AOP for your reference. If I have made any mistake, which is quite possible, please kindly point it out for me.    
Link to the full [DSP DPC code](https://github.com/pauloohaha/IWR6843AOPfullDspDpc/blob/main/xwr68xxFullDPC.zip):

I expect the reader to be familiar with the radar already to understand this document. I assume you have read the documents such as *mmwave_sdk_user_guide*, *introduction to the DSP Subsystem in the IWR6843*, the source code for the DPM, DPC, mmwaveDemo and so on.

## Usage
### CCS debug
Use the *xwr68xx_mmw_demo_mss.xer4f* and *xwr68xx_mmw_demo_dss.xe674* to load into the r4f and c674 in the CCS debug mode with MMWAVEICBOOST to run the code.  

### Compilation
Copy the *mmw_res.h* to the installed SDK path (mmwave_sdk_03_05_00_04\packages\ti\demo\xwr68xx\mmw) to replace the original *mmw_res.h* (remember to back up it). Then run the setenv.bat at *mmwave_sdk_03_05_00_04\packages\scripts\windows* though command line then cd to this folder and run *gmake clean* and *gmake all*.

More details can be viewed at the 4.5 section of the mmwave sdk userguide.  
## Intro
The out of box demos provided by the TI for IWR6843AOP use either full HWA or HWA+DSP data path to do the data processing, both of which uses HWA to do the range FFT. This prevent us from accessing the raw ADC buffer data directly, since the HWA destroy the ADC buffer data after FFT is done. Besides, DSP is also easier to control and program than HWA. Thus, realizing a full DSP data path is helpful for the development.  

However, TI only provide demo code that uses full DSP DPC for xWR16xx. We need to realise the full DSP DPC for IWR6843AOP on our own.  
  
In this document, I will introduce how to modify the out of box demo from mmave sdk 3.5 for IWR6843AOP to realize a full DSP DPC.  
  
## Basic
In the out of box demo from mmwave sdk 3.5 (mmwave_sdk_03_05_00_04\packages\ti\demo\xwr68xx\mmw), the demo uses HWA for range FFT and DSP for all other later data processing. This demo is a good template for programming both MSS and DSS, thus we will use it as a starting point.  

To modify it to a full DSP DPC demo, there a few changes we need to make:  
1. Disable the HWA intialization.  
2. Enable the range proc DSP.  
3. adjust the DPC for REMOTE mode.  
4. Set the correct resource allocation.  
  
## Disable the HWA intialization

### mmw_mss.h
Since now we do not use the HWA object detection, we change the  *#include <ti/datapath/dpc/objectdetection/objdetrangehwa/objdetrangehwa.h>* to *#include <ti/datapath/dpc/objectdetection/objdetdsp/objectdetection.h>* in mmw_mss.h
![image](https://user-images.githubusercontent.com/85469000/189802575-7ccd5c05-b90e-4902-98fe-eb64832b9b04.png)

### mmw_mss.mak
In *mmw_mss.mak*, we need to disable the reference to the HWA:  
At row 31, delete the reference for *librangeproc_hwa_xwr68xx.aer4f*.  
In the pictures below, the left is the modified and right is the original:  
![image](https://user-images.githubusercontent.com/85469000/189803098-37407bcc-1bdf-4da1-9124-e9cf5fa344ed.png)  

At row 82, delete the reference to the *objdetrangehwa.c*, thus it will not be included during compilation.:   
![image](https://user-images.githubusercontent.com/85469000/189803409-1626f56d-d2c4-4645-b3d5-852dead755a9.png)

At row 102, delete the marco defination of *OBJDET_NO_RANGE* to enable the code for DSP range processing. Remember to delete the entire line and make sure that there is not a empty line here.  
>![image](https://user-images.githubusercontent.com/85469000/189803463-d0a64bc6-a433-4edf-ac00-d018d067681a.png)
  
### mss_main.c
At the line 4077 and 4088 of *mss_main.c*, the *MmwDemo_initTask()* call the 2 functions for HWA init and open. Since we do not need HWA at all, we simply delet these 2 lines and the definations of these 2 functions.  
![image](https://user-images.githubusercontent.com/85469000/189802158-37375cf4-ae76-4385-a4c1-fed8ad022b5f.png)  
![image](https://user-images.githubusercontent.com/85469000/189820964-d2dbdb4e-c96b-4c03-86b9-e24dcf92a0e9.png)  
  
Therefore, change the DPC init parameters variable:  
>![image](https://user-images.githubusercontent.com/85469000/189802640-40d9a67f-81f1-4b61-b44e-87b9ae66c5ff.png)  

For DPM initialization, set the *ptrProcChainCfg* to NULL to disable DPC initialization at MSS. Set the *domain* to REMOTE, although this seems to make little difference. Set the *argSize* to the size of *objDetInitParams*, since we have changed its type earlier.  
>![image](https://user-images.githubusercontent.com/85469000/189803819-5987f5a2-1af1-46c6-ab69-69c2c5c60749.png)  

  
In *MmwDemo_dataPathConfig()*, we need to delete the DPM initialization of rangeProcHWA.  
From line 2023 to 2035, the code call DPM_ioctl to set the pre start common config the rangeProcHWA DPU, delete them.
>![image](https://user-images.githubusercontent.com/85469000/189804185-0d172d8a-20b8-4645-b52c-716bffda9e85.png)  

From line 2197 ti 2209, the code call MmwDemo_DPM_ioctl_blocking, which is basically a DPM_ioctl that halt the program before finish, to set the rangeProcHWA DPU pre start config, delete them.  
>![image](https://user-images.githubusercontent.com/85469000/189804429-2d1fc181-0b7b-42c0-9457-9a701f38d95c.png)  

Since now the *preStartCommonCfg* is never referenced, delete its declearation.  
>![image](https://user-images.githubusercontent.com/85469000/189804603-e12c96e4-f1f8-4d95-b25b-cafb28cabf5b.png)

In this way, we have disable all HWA initialization.  

## Enable the range proc DSP.  
### mmw_dss.mak
In mmw_dss.mak, we should include the reference to the rangeprocDSP. First, include the *librangeproc_dsp_xwr68xx.ae674* and its path:  
>![image](https://user-images.githubusercontent.com/85469000/189805009-6b91a890-5661-4677-9090-c8f6cb7fe838.png)  

Delete the defination of *OBJDET_NO_RANGE* at line 79 to enable code for the rangeProcDSP. Remember to delete the entire line and make sure that there is not a empty line here.
>![image](https://user-images.githubusercontent.com/85469000/189821243-f05b3e10-f457-4c0f-9dc1-7e2c2a00c53e.png)


### dss_main.c
The only change here is to set the domain to REMOTE mode.  
>![image](https://user-images.githubusercontent.com/85469000/189805217-8c426d5c-bdac-44c7-b750-b8749ff46d5c.png)

## adjust the DPC for REMOTE mode
### mss_main.c
*MmwDemo_DPC_ObjectDetection_reportFxn()* is responsible for handling the report message from DPM, including the *DPM_Report_DPC_STARTED*. In distributed mode, both the MSS and DSS will report start once, since they are responsible for HWA and DSP part of the DPC respectively. But in the remote mode, the MSS will not report start, since there is no DPC for it to start. DSS will report start once. Thus, at line 2572, we need to disable the checking for second start message:  
>![image](https://user-images.githubusercontent.com/85469000/189814834-ba97964d-0436-4990-b137-33772af51e7b.png)

## Set the correct resource allocation
EDMA instances and handles are allocated to each DPU in *mmw_res.h*. Since the changes are quit complex compared to the original HWA+DSP DPC, we directly use the *mmw_res.h* from the xWR16xx demo.  

In the *mmw_mss.mak* and *mmw_dss.mak*, the path to *mmw_res.h* is specified as the original path to the mmw_demo folder, we need to make change to the original *mss_res.h* file in the *mmwave_sdk_03_05_00_04\packages\ti\demo\xwr68xx\mmw* folder:  
![image](https://user-images.githubusercontent.com/85469000/189815596-bfc09b5c-e93b-4f84-a866-512c2edb31a1.png)

After copying the *mme_res.h* from xWR16xx demo folder to the xWR68xx folder, we need to add following defination into it. The IWR6843AOP has 2 EDMA channel controller, which is the 2 EDMA instance, one is allocated to the MSS and the other is allocated to the DSS. Since the DPC is realised entirly on DSP, the DPC EDMA instance uses the DSP EDMA instance.  
>![image](https://user-images.githubusercontent.com/85469000/189816003-6fedb497-f312-4d4a-9b6a-c353a8ee0418.png)

Since the LVDS driver is intialized on MSS, it should uses the MSS EDMA instance.
![image](https://user-images.githubusercontent.com/85469000/189816193-7865ca28-04e9-4a8b-b4de-a2c586330b26.png)

## Run the full DSP DPC
Normally the radar is set to send frames of chirps one after another. But in debug mode, the program is halt and may not be able to handle the chirp event. This will cause an error. Thus to debug the program, you need to set the cli command for *frameCfg* to send one frame only, which is setting the 4th arguement *number of frames* to 1. In this way, we can safely halt the program without causing any error.  

If you have sorted out all the bugs, you should see the following message in console, without any error messgae after it.  
![image](https://user-images.githubusercontent.com/85469000/189818260-2d0143fc-19ba-4bb3-a91c-56abdf52b541.png)


