Software Development Environment Setup
=======================================

Platform: Windows 7(x64),Windows 10(x64)

There are two types of toolchains we recommend to use for develpment:

#. Keil(MDK) + ARMCC + JLink
#. Eclipse + CMake + GCC + JLink

Keil(MDK) + ARMCC
###################

* MDK (Essential or Plus or Professional) version > 5.20  Download Link: https://www2.keil.com/mdk5 

Eclipse + CMake + GCC
#######################

* Download here: https://pan.baidu.com/s/1IEY80HunIiJmDUy1m08XDg  PASSWORD: g9s6 

* Follow these instructions:
        #. Create a new directory as your working directory (e.g.  *home*)
        #. Unzip eclipse_gcc_env.zip to directory  *home*
        #. Unzip SDK to directory  *home*, and rename SDK folder to  *Trunk* 
        #. Create a new directory  *gccbin* as CMake project directory in *home*
             We will get:
           ::
           
                    home
                    |----eclipse_gcc_env
                    |    |-----bin
                    |    |     |---...
                    |    |-----eclipse
                    |    |     |---...
                    |    |-----cmake-3.15.0-win32-w86
                    |    |     |---...
                    |    |-----gcc-arm-none-eabi-8-2019-q3-update-win32
                    |    |     |---...
                    |    |-----jre-8u121-windows-x64.exe
                    |    |-----readme.txt
                    |----Trunk
                    |    |-----CMakeLists.txt
                    |    |-----toolchain-gnu.cmake
                    |    |-----app
                    |    |     |---...
                    |    |-----tools
                    |    |     |---...
                    |    |-----ip
                    |    |     |---...
                    |    |-----modules
                    |    |     |---...
                    |    |-----plf
                    |          |---...
                    |----gccbin                    
                    
        #. Add the following path to STSTEM ENVIRONMENT PATH:
             home\\eclipse_gcc_env\\bin
           
             home\\eclipse_gcc_env\\cmake-3.15.0-win32-x86\\bin
           
             home\\eclipse_gcc_env\\gcc-arm-none-eabi-8-2019-q3-update-win32\\bin
        #. Go to directory *gccbin*, open a command line terminal in the directory. Execute the following command:
           ::
               cmake –G”Eclipse CDT4 – MinGW Makefiles” –DCMAKE_ECLIPSE_VERSION=4.5 –DCMAKE_ECLIPSE_MAKE_ARGUMENTS=-j –DCMAKE_TOOLCHAIN_FILE=../Trunk/toolchain-gnu.cmake ../Trunk            
        #. The project has been successfully generated.Try to compile an example program in command line:
           ::
               cmake --build . --target osapp_dis_server -- -j
           
           Or import this project into Eclipse, then click Build Targets->Targets->osapp_dis_server->Build  
        #. After the building process,go to directory *gccbin\\output\\osapp_dis_server* for outcome

JLink
######

* JLink Software Version > 6.16a Download Link: https://www.segger.com/downloads/jlink/









