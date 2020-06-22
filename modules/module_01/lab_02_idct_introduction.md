## Becoming Familiar With IDCT Application

This lab is designed to teach the fundamentals of the Vitis development environment and programming model. Its contents are tailored, to familiarize user with basic OpenCL APIs, understanding of software and hardware emulation flows, profiling performance and identifying how to optimize host code and kernel hardware implementation.

The kernels or the functions used for FPGA acceleration in this lab are slightly different hardware implementations of Inverse Discrete Cosine Transform (IDCT) algorithm, a function widely used for transform coding in applications like audio/image codecs such as JPEG and High Efficiency Video Coding(HEVC).


### Setting Up Vitis Environment

1.  Open a new terminal by right-clicking anywhere in the desktop area and by selecting **Open Terminal**.

1.  Set up the Vitis environment.  

    ```bash
    source $AWS_FPGA_REPO_DIR/vitis_setup.sh
    ```
    
	*Note: the Vitis_setup.sh script might generate warning messages, which can be safely ignored.*

1.  Go to design folder and investigate the design files.

    ```bash
    # Go to the lab directory
    export LAB_WORK_DIR=/home/centos/src/project_data/
    cd $LAB_WORK_DIR/Vitis-AWS-F1-Developer-Labs/modules/module_01/idct/
    ls
    ```
	  The `src` folder contains the kernel source files and host code. The `Makefile` is provided for design compilation and execution. Let's open up the make file and take a look at it.
    ```
    vi Makefile
    ```

    In the 'platform selection' section, the default target platform is set as `xilinx_aws-vu9p-f1_shell-v04261818_201920_1` which is the AWS F1 platform.

    The next couple of lines define the design files location and file names. Following that is the host compiler settings, kernel compiler and linker settings.  You don't need to modify any of the options here but you may want to play with them after finishing this tutorial.

    Exit **Makefile** view window and let's take a look at the design files.

### Overview of the Application Source Code

---
The project comprises of multiple files under src directory, following is the list with brief description:

* **host.cpp** contains main function, performs data initialization, allocation and forks CPU and FPGA threads 

* **krnl_idct_wrapper.hpp** contains **_runFPGA_** thread that coordinates kernel execution on FPGA accelerator  

* **idct.cpp** contains the software implementation of IDCT (reference model) that will run as a separate thread on CPU or host side.

* **krnl_idct.cpp** contains the code for IDCT kernel (custom accelerator) for running it on FPGA.

* **krnl_idct_med.cpp** contains a version of IDCT kernel for demonstration purposes only **_with II=4_**

* **krnl_idct_noflow.cpp** contains a version of IDCT kernel for demonstration purposes only **_without dataflow optimization_** 

* **krnl_idct_slow.cpp** contains a version of IDCT kernel for demonstration purposes only **_with II=8_**

* **types.hpp** defines IDCT I/O and DDR interface types

* **utilities.hpp** contains utility function used for printing execution information to console 
---

2.  Open the **krnl_idct.cpp** file to see the code structure used for **IDCT kernel**.
	* The **idct** function is the core algorithm implemented as a custom hardware accelerator.
	* This computationally heavy function can be highly parallelized on FPGA using a parallel datapath providing significant acceleration compared to CPU-based implementation.
	* The **krnl_idct** function is the top-level for the custom hardware accelerator. Memory interface properties for the accelerator are specified in this function allowing us to utilize the maximum potential bandwidth of device DDR memory. 
	* The **krnl_idct_dataflow** function is called by the top-level function and encapsulates the main functions of the accelerator.
	* The **read_blocks** function reads data from device DDR memory written by the host application and streams(writes to an output FIFO storage) this data to the **execute** function.
	* The **execute** function receives the streaming data and, for each 8x8 block received, calls the **idct** function to perform the actual computation and streams the results to and output FIFO.
	* The **write_blocks** function receives the streaming results from the **execute** function through a FIFO and writes them back to device DDR memory which are read by the host application.
	
3. The kernels in the source files **krnl_idct_slow.cpp** , **krnl_idct_med.cpp** and **krnl_idct_noflow.cpp** describe very similar kernels. These are almost identical but with variations on micro-architecture for hardware implementation. Which can be achieved by adding and removing some **Vitis HLS pragmas**. These kernels are designed to *show how different micro-architectural variations can be used to trade-off FPGA hardware resources with performance*. In this lab we will do all experiments related to kernel **"krnl_idct"** only. 

4. Open the **host.cpp** file and observe following sequence of steps carried out by host application:  
	* **Command line arguments:** The **main** first parses the command line arguments. These command line arguments can be used to control total number of IDCT blocks to be processed and how many IDCT blocks kernel processes in one go (call made from host side with appropriate data size) also called batch size.

    * **Test Vectors:** next test vectors are allocated and initialized

    * **Device Programming:**  In next phase Xilinx device search is performed and the first found device is programmed with user provided Xilinx binary also called AFIs for AWS. An associated OpenCL Context and Command Queue are also created.

    * **Kernel Handle:** after the device is programmed with FPGA Image a kernel handle is created which can be used to process data.

    * **CPU and FPGA Threads:** next host launches two separate threads **runCPU** and **runFPGA** for CPU run and FPGA accelerated run respectively at the same time.

    * **Execution Time Measurement:** Once cpu and FPGA thread are forked, main thread calls **measureExecTimes** function which waits on these threads to finish and also samples the time when these threads finish, to calculate execution times. 
    * **Output Validation and FPGA Peformance:** Once threads finish execution FPGA results are validated by comparing against reference output generated by CPU run and performance comparison is printed out. The performance numbers give metrics such as execution times and throughput(data processing rates).

 9. Now go to **host.cpp** line 47 three important parameters are defined here **BATCH_SIZE**, **NUM_OF_BATCHES** and **MAX_SCHEDULED_BATCHES**:
 
    * **BATCH_SIZE**: Defines number of IDCTs carried out by single call to the kernel, it can also be specified at runtime as host application command line argument
    * **NUM_OF_BATCHES**: Defines number of batches to be processed in total by the application
    * **MAX_SCHEDULED_BATCHES**: If multiple batches of input data are to be processed then there is a potential for IDCT compute and data transfer to be overlap for different batches, this parameter specifies how many batches can be scheduled to be processed in overlapping fashion, essentially it defines number of duplicate resources such as **cl::Buffer** objects.

7. Go to label **FPGA_DEVICE_OPENCL_SETUP** near line 186 of the **host.cpp** file: 
    
    This section sets up OpenCL environment for **runFPGA** thread. It is very typical setup for Vitis acceleration applications very familiar to developers with prior OpenCL experience. Following section provides the briefs of used APIs. 
   * **xcl::get_xil_devices**: Xilinx provided API, returns a list of Xilinx devices connected to the host
   * **xcl::read_binary_file**: Xilinx provided API, reads a compiled binary file for FPGA
   * **cl::Program::Binaries**: Creates a binary file object from raw binary file which can be used to create a OpenCL program  associated with a device, essentially programming FPGA device there in.
   * **cl::Program**: Creates a cl::Program objects and also programs FPGA device. The programmed device may have multiple kernels compiled inside single programs so created object also provides a reference that can be used to create  handles to different kernels.
   * **cl::Kernel**: Creates a kernel object given the cl::Program handle and kernel name as string, in next labs we will change this name string often, in this case we have multiple kernels but here we are going to use one kernel namely "krnl_idct".
   
   **NOTE**:_Many function calls and object construction call use a macro **OCL_CHECK** which is used to parse the return status of type **cl_int**. It errors out after OpenCL call, if call doesn't complete as expected._


5. Open the **krnl_idct_wrapper.hpp** file.
    * This file defines a thread wrapper function for the use of host side, it has the host side code that runs on CPU and interacts with the FPGA accelerator using OpenCL C/C++ APIs. The host code structure and the use of OpenCL C/C++ APIs is very typical and similar to the one used in any other heterogeneous acceleration framework such as the ones based on GPUs. 
    
    * The **runFPGA** wrapper function takes as input: 
        * data size
        * batching info
        * OpenCL objects like command queue, context and kernel 
        * input data to be transformed passed as a vector
        * input vector for IDCT coefficients
        * an output vector.

    Now see how **runFPGA** performs IDCT compute using FPGA kernel which is very similar to any other device kernel such as GPU used for compute with OpenCL APIs. The basic logic behind this pieces of code is as follows:

    * It creates vector of events to define the dependencies between different compute and data migration tasks such as: 
        
        * kernel compute should be triggered only when host to device data transfer completes

        * Output data transfer from device back to host should trigger only when kernel compute completes

        * next batch of input data(triggering processing of next batch of data) transfer from host to device should wait for current transfer
       
    Next **cl::Buffers** objects for **input**,**output** and **coefficients** are created that will be used to transfer data from host to device (FPGA) and back. The main loop runs over number of batches to be processed and inside this loop **cl::Buffer** objects are initialized with proper attributes and host pointers, since the input data is contiguous(stored as single block of multiple input batches) so host pointer is provided by calculating the offset based on current batch number and batch size over the input data. Host side application uses number of **cl::Buffer** object each for input and output data, this number can be selected by passing command line argument to host application. These set of buffers are used in a circular buffer style to create a pool of simultaneously enqueued task which allows compute and data transfers to overlap between different kernel calls.

    * **setArg**: is used to set different kernel arguments.

    * **enqueueMigrateMemObjects**: is used to enqueue data transfer request between host and FPGA device.

    * **enqueueTask**: is used to enqueue kernel on command queue for execution, it takes as input list of events that should be completed and also produces an event to signal end of execution.         
    
 6. Open the file **idct.cpp**

	* The **runCPU** function is defined here it takes a vector of inputs and, for each 8x8 block, calls **idctSoft**, a reference implementation of the IDCT to be ran on CPU.

	* The **idctSoft** function is the reference software implementation of the IDCT algorithm, used in this example to check the results coming back from the FPGA.


All of the OpenCL API functions used here are documented by the [Khronos Group](https://www.khronos.org), the maintainers of OpenCL, the open standard for parallel programming of heterogeneous systems.


### Running the Application
  Vitis applications can run in multiple modes, these modes include software emulation, hardware emulation and actual system run on FPGA acceleration card. Vitis provides two emulation flows which allow testing the application before deploying it on the F1 instance or actual FPGA accelerator. These flows are referred to as software emulation and hardware emulation modes.

  * **Software emulation** is used to identify syntax issues and verify the behavior of application in this mode the C/C++ model of kernel is used for execution instead of any hardware or hardware model.
  * **Hardware emulation** is used to get performance estimates for the accelerated application with more realistic results than software emulation flow but it can only provide very accurate estimates about the hardware accelerated function/kernel performance and FPGA resource usage, the memory interfaces and data transfers use approximate models only. In this mode RTL model is used for kernel.
  

#### Running Software Emulation

1. Run below commands in the terminal window.
    ```bash
    source $AWS_FPGA_REPO_DIR/vitis_setup.sh
    cd $LAB_WORK_DIR/Vitis-AWS-F1-Developer-Labs/modules/module_01/idct/
    make run TARGET=sw_emu
    ```
    This will run through software emulation and print out messages as shown below to indicate that the software emulation process has finished successfully. Output log will signal the execution mode such as sw_emu/hw_emu or system hw. The emulation mode is defined by environment variable **XCL_EMULATION_MODE** set by makefile which is read by host executable to identify run mode, host application tries to identify run mode, to change input data size for quick execution in case of emulation mode.

    ```bash
    ------ Identified Run mode : sw_emu
    .
    .
    .
	=====================================================================
    ------ Launched CPU and FPGA Thread and Monitoring Execution.
    =====================================================================
    [FPGA Time(     1s ) : ]  [CPU Time(    1s ) : ]
    Execution Finished
    =====================================================================
    ------ All Task finished !
    ------ Done with CPU and FPGA based IDCTs
    ------ Runs complete validating results
    ------ TEST PASSED ------
    ```

    The generated files are put into `build` folder under `design` directory. You can use `ls` command to investigate the generated files.

#### Running the Hardware Emulation
 
1. After software emulation finishes successfully, you can move forward and run the design in hardware emulation mode. The corresponding command is:
    ```bash
    cd $LAB_WORK_DIR/Vitis-AWS-F1-Developer-Labs/modules/module_01/idct/
    make run TARGET=hw_emu
    ```

	* In hardware emulation, the host code is compiled to run on the x86 processor and the kernel code is compiled into a hardware model (known as RTL or Register Transfer Level) which runs in an RTL simulator.
	* The build and run cycle takes longer because the kernel code is compiled into a detailed hardware model which is slower to simulate.
	* The more detailed hardware simulation allow more accurate reporting of kernel and system performance.
	* This flow is also useful for testing the functionality of the logic that will go in the FPGA.
	* The hardware emulation is complete when the following messages are displayed:

    ```bash
	=====================================================================
	------ Launched CPU and FPGA Thread and Monitoring Execution. 
	=====================================================================

	[FPGA Time(    21s ) : ]  [CPU Time(    1s ) : Done. ]  

	Execution Finished
	=====================================================================
	------ All Task finished !
	------ Done with CPU and FPGA based IDCTs
	------ Runs complete validating results
	------ TEST PASSED ------
	INFO::[ Vitis-EM 22 ] [Time elapsed: 1 minute(s) 19 seconds, Emulation time: 0.132265 ms]
	Data transfer between kernel(s) and global memory(s)
	krnl_idct_1:m_axi_gmem0-DDR[0]                 RD = 32.000 KB              WR = 0.000 KB        
	krnl_idct_noflow_1:m_axi_gmem1-DDR[1]          RD = 0.000 KB               WR = 0.000 KB      
	krnl_idct_noflow_1:m_axi_gmem2-DDR[2]          RD = 0.000 KB               WR = 0.000 KB      
	krnl_idct_slow_1:m_axi_gmem0-DDR[0]            RD = 0.000 KB               WR = 0.000 KB        
	krnl_idct_slow_1:m_axi_gmem1-DDR[1]            RD = 0.000 KB               WR = 0.000 KB        
	krnl_idct_slow_1:m_axi_gmem2-DDR[2]            RD = 0.000 KB               WR = 0.000 KB        
	krnl_idct_1:m_axi_gmem1-DDR[1]                 RD = 1.000 KB               WR = 0.000 KB        
	krnl_idct_1:m_axi_gmem2-DDR[2]                 RD = 0.000 KB               WR = 32.000 KB       
	krnl_idct_med_1:m_axi_gmem0-DDR[0]             RD = 0.000 KB               WR = 0.000 KB        
	krnl_idct_med_1:m_axi_gmem1-DDR[1]             RD = 0.000 KB               WR = 0.000 KB        
	krnl_idct_med_1:m_axi_gmem2-DDR[2]             RD = 0.000 KB               WR = 0.000 KB        
	krnl_idct_noflow_1:m_axi_gmem0-DDR[0]          RD = 0.000 KB               WR = 0.000 KB 
    ```
   
**NOTE:** Vitis Emulation info message "INFO::[ Vitis-EM 22 ]" lists interfaces used by all the kernels and data transferred over them, since in this experiment only kernel "krnl_idct" is used by host so only data transfers for this kernel have non-zero values. Also during the emulation host application identified the emulation mode and used a small data size to ensure short emulation time. To do the full length emulation, please use makefile as follows: 
 ```bash
make run TARGET=sw_emu EMU_FULL=1 BATCH_SIZE=1024 NUM_OF_BATCHES=16 MAX_SCHEDULED_BATCHES=8
 ```   
Here you can choose the batch size, number of batches , max scheduled batches and full length emulation mode by setting **EMU_FULL=1**, if it is set to '0' the emulation will still use small data set for emulation. For actual FPGA run full length data will be used. In general the setting of these parameter be such that:

   * **NUM_OF_BATCHES** = N * **MAX_SCHEDULED_BATCHES**
   
   *where N is an integer number* 
   
### Summary 
In this lab you learned:
* About the IDCT application source code structure
* OpenCL C/C++ APIs used to communicate between Host and FPGA
* How to run IDCT application in different Emulation Modes
* Differences between software and emulation flows

---------------------------------------

<p align="center"><b>
Start the next lab: <a href="lab_03_vitis_analyzer.md">Application Performance Analysis</a>
</b></p>  