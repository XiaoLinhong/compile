### 前言

文档描述常见的气象、空气质量模型安装方式（WRF、WRF-Chem、CAMX、CMAQ）。

模型安装之前，需要提前编译好模型的依赖库，详情见[模式中台环境搭建](http://47.92.132.84:3000/bedrock/envs)。


**通常我们所说的模型安装** 是指程序的编译过程。我们常接触到的模型，大多采用FORTRAN、C、C++等编译型语言进行编写。

编译型语言的特点（相对于解释型语言，如bash、python）是 在执行之前，需要通过编译器将源代码（本文格式）*翻译*成二进制（linux下面为 ELF文件格式）！

编译型语言编写的大型程序一般都会依赖一些基础库（线性代数、IO格式（要感谢各种好用的格式出现）、压缩算法等），因而在编译的时候，需要让编译器找到相应的东西。

这里首先需要明确的是，编译过程其实是包含了两个分步骤（可以分解更多，暂时不讨论）：

* **编译**（compile，将源代码翻译成目标文件，object file，也就是.o文件（也是ELF格式），由编译器compiler完成）
* **链接**（link，将目标文件、静态库、动态库链接成一个可执行文件，由连接器linker完成）。

由于大多数编译型语言都是强类型语言（且变量使用之前需要声明），在编译的时候，如果依赖第三方库，通常需要第三方库声明（记住使用变量之前必须声明）的头文件（C语言系列的include，.h文件，FORTRAM语言的模块，.mod文件）。同时在链接的时候，需要让链接器找到静态库、动态库等位置（相关路径）。所以配置依赖库，是指配置**头文件的路径和库文件的路径**。

**编译**命令如下（注意-I的作用）：
```
/opt/hpc/software/mpi/hpcx/v2.4.1/gcc-7.3.1/bin/mpif90 -c -o Mod_src/node_mod.o \
-I./Includes -I/opt/hpc/software/mpi/hpcx/v2.4.1/gcc-7.3.1/include  \
-I/public/software/mathlib/netcdf/4.4.1/gcc/include -O2  \
-J./Modules -fno-align-commons -fconvert=big-endian -frecord-marker=4 -ffixed-line-length-0 -fopenmp  \
-I./Modules Mod_src/node_mod.f
```

**链接**命令如下（注意-L的作用）：
```
/opt/hpc/software/mpi/hpcx/v2.4.1/gcc-7.3.1/bin/mpif90 -o CAMx.v6.50.openMPI.NCF4.ieee.gfortranomp \
.... # 省略
./MPI/nodes_send_puffmass_back.o ./MPI/nodes_pass1.o \
./MPI/mpi_feedback.o ./MPI/par_decomp.o ./MPI/par_decomp_bounds.o \
-L/public/software/mathlib/netcdf/4.4.1/gcc/lib -lnetcdff -lnetcdf -L./MPI/util -lparlib  
```

可以想象，对于大型程序而言，一个一个的目标文件手动编译，最后再进行链接，将成为一种灾难！ 为了简化这个过程（这就是程序员的思路，一切程序化），引入了一个管理编译过程的工具，[GNU Make](http://www.gnu.org/software/make/)（GNU Make is a tool which controls the generation of executables and other non-source files of a program from the program's source files.）。Make的工作内容是通过Makefile去描述的（有自己的语法规则，可以将Make看作一个解释器，makefile就是解释语言的源代码）。一些模式选择将这种方式直接暴露给下游用户（flexpart、CAMx等，直接让用户修改Makefile），这就需要用户自己能够看懂makefile的语言，根据自己的环境（操作系统版本、编译器、编译器版本、编译器选项、库的路径等），做适当的修改，这一般对于新手不太友好。

常见的make命令：
```
make -f makefile install
make install
make clean
make all
```

为了避免直接将makefile的语法直接暴露给用户，基于make工具，社区开发了[autotool](https://www.gnu.org/software/automake/manual/html_node/Autotools-Introduction.html)帮助大家去配置自己的编译环境，很多标准的库，都选者了这种方式（比如Netcdf，zlib等）。也就是说到了这个阶段，基本的编译局面是make控制编译过程， autotool控制环境的切换。

常见的autotool编译流程：
```
./configure --help # 可以使用这个命令查看有哪些环境可以设置（命令行的方式）
./configure --prefix=/your/out/path # 这一步就是为了生成makefile，避免自己去改变makefile
make # 可以不要
make install
```
一些模式也选择其他方式去隐藏makefile，比如使用shell（如CMAQ等），大致的思路和autotool基本一致，不过自己需要去改shell脚本（可能是默认大家都会shell吧）

autotool发展后期大家觉得这维护成本太高，编程也不是很麻烦，感觉不太象一个程序语言，后期就发展了cmake这个工具，比如GSI目前就是基于cmake编译的，cmake的作用和autotool一样，都是用来生成makefile的，解决环境配置问题，特别是依赖库的检索。
```
cmake ../
make # 可以不要
make install
```

无论是shell、还是autotool、还是cmake，都存在通过环境变量和命令行两种方式去获取外部的配置信息，因此如果要掌控住大多数的模式编译，需要熟悉各种编译器（intel、gnu、mpi等）、各种shell环境（bash、csh）、makefile、autotool、cmake等，还需要了解编译的过程和执行过程（loader，加载器，不然找不到动态库会一脸懵逼！）。

当然除了这种手动编译的过程，还有很多包管理器可以用来直接安装一些库和程序，像rpm、yum等，大大的简化了依赖库的安装过程，不过一般需要管理员权限，一些新兴的编译型程序也效仿解释性语言（比如python的pip、js的npm等）可以拥有自己的包管理器软件（比如rust的cargo），这样就完全摆脱了makefile这种战国时代的工具！

### WRF编译
正常编译过程：
```
export HDF5=$ENVS/hdf5/1.10.5 # 安装WRF需要
export NETCDF=$ENVS/netcdf/4.7.0 # WPS安装需要
export JASPERLIB=$JASPER/lib # 安装WRF需要
export JASPERINC=$JASPER/include # 安装WRF需要
./configure
./compile em_real >& log
export J='-j 6'; ./compile em_real >& log 并行安装
```

可以通过修改configure.wrf，去改变编译器和一些选项（要明白自己用的是啥编译器，啥库！）
``` 
vim configure.wrf
DM_CC  = mpiicc
DM_FC  = mpiifort
```

### WRFC编译

WRFC特有的环境变量设置（通过环境变量将参数传递给configure）
```
export EM_CORE=1 
export WRF_CHEM=1
export FLEX_LIB_DIR=/usr/lib
export YACC='/usr/bin/yacc -d'

```
基本和WRF一致，正常编译过程：
```
export HDF5=$ENVS/hdf5/1.10.5 # 安装WRF需要
export NETCDF=$ENVS/netcdf/4.7.0 # WPS安装需要
export JASPERLIB=$JASPER/lib # 安装WRF需要
export JASPERINC=$JASPER/include # 安装WRF需要
./configure
./compile em_real >& log
export J='-j 6'; ./compile em_real >& log 并行安装
```

``` 
vim configure.wrf
DM_CC  = mpiicc
DM_FC  = mpiifort
```

### CAMX编译
```
 # 修改makefile和MPI/util/Makefile
 # MPI_INST = /public/home/bedrock/envs/v1.0/mvapich/2-1.9
 # NCF_INST = /public/home/bedrock/envs/v1.0/netcdf/4.7.0
 make COMPILER=ifortomp CONFIG=v6.50 MPI=MVAPICH NCF=NCF4_C IEEE=true
```

### CMAQ编译
第一步拷贝编译环境
```
vim bldit_project.csh 
set CMAQ_HOME = /public/home/bedrock/pkgs/apps/CMAQ
./bldit_project.csh
```
第二步进入编译环境
```
vim config_cmaq.sh 
setenv IOAPI_INCL_DIR   /public/home/bedrock/envs/v1.0/ioapi/3.2/ioapi/fixed_src    #> I/O API include header files
setenv IOAPI_LIB_DIR    /public/home/bedrock/envs/v1.0/ioapi/3.2/lib    #> I/O API libraries
setenv NETCDF_LIB_DIR   /public/home/bedrock/envs/v1.0/netcdf/4.7.0/lib   #> netCDF C directory path
setenv NETCDF_INCL_DIR  /public/home/bedrock/envs/v1.0/netcdf/4.7.0/include   #> netCDF C directory path
setenv NETCDFF_LIB_DIR  /public/home/bedrock/envs/v1.0/netcdf/4.7.0/lib  #> netCDF Fortran directory path
setenv NETCDFF_INCL_DIR /public/home/bedrock/envs/v1.0/netcdf/4.7.0/include  #> netCDF Fortran directory path
setenv MPI_LIB_DIR  /public/home/bedrock/envs/v1.0/intel/2019.4/compilers_and_libraries_2019.4.243/linux/mpi/intel64 
 ./config_cmaq.csh intel
```

第三步编译mcip
```
 # 修改makefile：配置netcdf的路径
 make 
```

第四步编译icon
```
./bldit_icon.csh intel
```

第五步编译bcon
```
./bldit_bcon.csh intel
```

第六步编译cctm
```
vim bldit_cctm.csh # 修改化学机制
set Mechanism = cb6r3_ae6_aq
./bldit_cctm.csh intel
```

### GSI

注意gsi不能使用intel2019以上的版本编译

``` bash
source /public/software/intel/parallel_studio_xe_2018/bin/compilervars.sh intel64
source /public/software/intel/parallel_studio_xe_2018/impi_latest/intel64/bin/mpivars.sh

export NETCDF_DIR=/public/home/bedrock/envs/v1.0/netcdf/4.7.0
export CMAKE_C_COMPILER=icc
export CMAKE_CXX_COMPILER=icpc
export CMAKE_Fortran_COMPILER=ifort

cd /public/home/tangbx/work/software/GSI

mkdir build; cd build
cmake ../comGSIv3.7_EnKFv1.3

make -j8
```

### 后记

在实践过程，可能会遇到各种各样的错误，不要慌张，机器相关的虽然庞杂（穷其一生，也无法达到全知全能的状态），但是它的逻辑是简单的，是固定的，面对它们，不应该有任何畏难情绪（因为它简单）。

本文档介绍的是我在模式安装领域建立的**认知系统**，读完整个文档，也不一定在编译的时候一帆风顺（环境是多变的），但是希望能够帮助大家在这个领域去构建一个自己**认知系统**，在需要处理相应工作时，因该基于已经构建的认知系统去思考处理思路，遇到问题的时候，应该基于认知系统去解决问题（而不是瞎搞、瞎试）。当你的认知系统无法解决该领域的问题时，或者无法高效地解决该问题时（我们都通常喜欢待着自己的舒服区，也可能是学习成本的原因，只要能解决，不愿意去改变，不过希望大家能够能多的勇气、决心和魄力去探索！），就是它升级、扩充的时候，也是你成长的时候（应该感到开心、而不是沮丧）。