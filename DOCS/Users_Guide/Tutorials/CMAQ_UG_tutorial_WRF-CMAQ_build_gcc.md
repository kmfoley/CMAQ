## WRF-CMAQ Tutorial ## 
This  tutorial provides instructions for contructing the WRF-CMAQ model with WRF version 4.4 or later and CMAQ version v5.3.3+. Directions for constructing the WRF-CMAQ model with WRF version 4.3 and CMAQv5.3.3 can be found in the [CMAQv5.3.3 WRF-CMAQ tutorial](https://github.com/USEPA/CMAQ/blob/main/DOCS/Users_Guide/Tutorials/CMAQ_UG_tutorial_WRF-CMAQ_build_gcc.md).


### Procedure to build the WRF-CMAQ model using gnu compiler: ###

### Step 1: choose your compiler, and load it using the module command if it is available on your system

```
module avail
```

```
module load openmpi_4.0.1/gcc_9.1.0 
```

### Step 2: Download and install netCDF Fortran and C libraries

   **Skip to Step 3, if you have a module for netCDF avialable on your system and you have loaded it**

   Follow the tutorial for building libraries to build netCDF C and Fortran Libraries
   https://github.com/USEPA/CMAQ/blob/main/DOCS/Users_Guide/Tutorials/CMAQ_UG_tutorial_build_library_gcc.md
   
   - Follow these instructions to combine the libraries into a single combined directory
   
   ```
   cd /[your_install_path]/LIBRARIES
   mkdir netcdf_combined
   cp -rp ./netcdf-fortran-4.4.5-gcc9.1.0/* ./netcdf_combined/
   cp -rp ./netcdf-c-4.7.0-gcc9.1.0/* ./netcdf_combined/
   ```
   
   Now you should have a copy of both the netcdf C and netcdf Fortran libraries under 
   netcdf_combined/lib

   - set the following environment variables including the path to your combined netcdf libraries, include files
   
   ```
   setenv NETCDF [your_install_path]/LIBRARIES/netcdf_combined
   setenv CC gcc
   setenv CXX g++
   setenv FC gfortran
   setenv FCFLAGS -m64
   setenv F77 gfortran
   setenv FFLAGS -m64
   ```
   
 - check to see that the path to each compiler is defined using
 
    ```
    which gcc
    which g++
    which gfortran
    ```
    
  - If they are not found, ask for assistance from your system administrator, 
    or if you know the path then specify it using the environment variable
    
    ```
    setenv CC /nas/longleaf/apps/gcc/9.1.0/bin/gcc
    ```

### Edit your .cshrc to add the path to the library by setting the LD_LIBRARY_PATH environment variable

```
#for gcc WRF-CMAQ build
setenv NCF_COMBO /[your_install_path]/LIBRARIES/netcdf_combined/
setenv LD_LIBRARY_PATH ${NCF_COMBO}/lib:${LD_LIBRARY_PATH}
```

### Make sure that there is no other definition or setting of LD_LIBRARY_PATH further down in your .cshrc file that may be overwriting your setting.

### Make sure you log out and log back, or run csh in to activate the LD_LIBRARY_PATH setting.

### Step 3: Download IOAPI_3.2 (a specific tagged version, see below) and install it.

Note The complete I/O API installation guide can be found at either of the following:

https://www.cmascenter.org/ioapi/documentation/all_versions/html/AVAIL.html

or

https://cjcoats.github.io/ioapi/AVAIL.html

#### Follow the instructions on how to install I/O API available

#### Method 1. Download the tar.gz file from the github site.

     cd /[your_install_path]/LIBRARIES     
     wget http://github.com/cjcoats/ioapi-3.2/archive/20200828.tar.gz
     tar -xzvf 20200828.tar.gz
     cd ioapi-3.2-20200828
     

#### Method 2. Use Git clone to obtain the code
    
     cd /[your_install_path]/LIBRARIES 
     git clone https://github.com/cjcoats/ioapi-3.2
     cd ioapi-3.2         ! change directory to ioapi-3.2
     git checkout -b 20200828   ! change branch to 20200828 for code updates
     cd ..                      ! change directories to the level above ioapi-3.2
     ln -s ioapi-3.2 ioapi-3.2-2020828 ! create a symbolic link to specify the tagged version
     cd ioapi-3.2                      ! change back to the directory
     

#### Change directories to the ioapi directory
     
     
     cd ioapi
     
     
#### Copy the Makefile.nocpl to Makefile 
     
     
     cp Makefile.nocpl Makefile

#### Change the BASEDIR definition from HOME to INSTALL

```
BASEDIR = ${HOME}/ioapi-3.2
````
change to
```
BASEDIR = ${INSTALL}/ioapi-3.2
```
     
     
 #### set the INSTALL and BIN environment variables:
     
     
     setenv INSTALL [your_install_path]/LIBRARIES
     setenv BIN  Linux2_x86_64gfort_openmpi_4.0.1_gcc_9.1.0
     
     
 ### set the CPLMODE environment variable
 
     setenv CPLMODE nocpl
     

 #### Make the installation directory

    
     mkdir $INSTALL/$BIN

 ### Link the installation directory to the generic directory name supported by WRF-CMAQ 
 
     cd $INSTALL
     ln -s Linux2_x86_64gfort_openmpi_4.0.1_gcc_9.1.0 Linux2_x86_64gfort
      
 ### Edit the Makefile to add a path to the combined netCDF library directory
 ### Note this is the Makefile at the ioapi-3.2 level. 
 ### First need to copy Makefile.template Makefile
 
 ```
 cp Makefile.template Makefile
 ```
 
 change
 
 ```
 NCFLIBS = -lnetcdff -lnetcdf
 ```
 
 to
 
   ```
   NCFLIBS    = -L $NETCDF/lib/ -lnetcdff -lnetcdf   ! using the combined $NETCDF environment variable set above
   ```
 
 #### change into the ioapi directory and copy the existing Makeinclude.Linux2_x86_64gfort to have an extension that is the same as the BIN environment variable
 
 ```
 cd ioapi
 cp Makeinclude.Linux2_x86_64gfort Makeinclude.Linux2_x86_64gfort_openmpi_4.0.1_gcc_9.1.0
 ```
 ### Edit the Makeinclude.Linux2_x86_64gfort_openmpi_4.0.1_gcc_9.1.0 to comment out the OMPFLAG and OMPLIB
 
 ```
 gedit Makeinclude.Linux2_x86_64gfort_openmpi_4.0.1_gcc_9.1.0
 ```
 
 - comment out the following lines by adding a #
 
 ```
# OMPFLAGS  =  -fopenmp
# OMPLIBS   =  -fopenmp
 ```
 
 ### Create the Makefile in the m3tools directory
 
 ```
 cd ../m3tools
 cp Makefile.nocpl Makefile
 ```
 
 
 ### Build ioapi using one of the following commands
 
 ```
 cd ioapi 
 make HOME='[your_install_path]/LIBRARIES' |& tee make.log ! method if you did not modify HOME variable in Makefile
 or
 make |& tee make.log     ! method if you did replace HOME variable with INSTALL
 ```
 
 ### Verify that the libioapi.a and the m3tools have been successfully built
 
 ```
 ls -lrt $INSTALL/ioapi-3.2-20200828/Linux2_x86_64gfort_openmpi_4.0.1_gcc_9.1.0/libioapi.a
 ```
 
 ### Note: If you get a shared object problem when trying to run m3tools such as the following:
 ```
./juldate
./juldate: error while loading shared libraries: libimf.so: cannot open shared object file: No such file or directory
```
### Be sure that the appropriate module is loaded, or that the LD_LIBRARY_PATH contains a path to the shared opject file that is missing.
```
module load openmpi_4.0.1/gcc_9.1.0
```

 ### Note: If you need to rebuild the I/O API library to remove the dependency on OpenMP use
 ```
 cd ioapi
 make HOME='[your_install_path]/LIBRARIES' clean ! if you have not modified the Makefile
 or 
 make clean ! if you have modified the Makefile to use the INSTALL environment variable and set it at the command line
 ```

### Step 4: Download WRFv4.4

### Step 5: Download s CMAQ 5.3.3+ version and build a CMAQ model with coupled model function turned on

### Step 6: Move that CMAQ model in the WRF code directory and rename it as a subdirectory, cmaq

### Step 7: Setup appropriate environment variables by typing 
            setenv WRF_CMAQ 1

            setenv IOAPI the_explicit_path_of_the_ioapi_library

            setenv WRFIO_NCD_LARGE_FILE_SUPPORT 1  (this is optional but it is good to have)

### Step 8: Go through regular WRF building and compiling processes by typing
            configure
            compile em_real
