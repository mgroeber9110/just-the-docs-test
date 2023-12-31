## 6 Debug Utility

The Debug application allows the user to quickly change the debugging 
environment on the target machine. It works with the target's .ini to simulate 
the UI of different kinds of machines. It also allows for an easy way to switch 
back and forth between the regular and profiling kernels.

*Warning:*  
Debug's simulation simulates the UI layouts of various platforms. While this 
is useful for laying out an application's UI gadgetry, remember that the real 
platforms will have different sets of system files and different hardware. For 
more complete testing, you should use an environment with the complete 
system files (e.g., to test a Zoomer program, use the ZOOM target directory). 
If your program is meant to work on a particular type of device, you should 
run some tests on that device as well.

### 6.1 Changing Platforms

Select the "Platform" item of the Options menu to change to another 
platform. Debug allows the target machine to simulate different platforms by 
means of .ini files which are stored in the target machine's INI subdirectory 
of the top-level GEOS directory. Depending on the contents of the selected .ini 
file, GEOS will act like a different device.

To select a new .ini file, click on it. A description of the platform simulated by 
the .ini file will appear. Some .ini files have been set up to simulate special 
hardware devices; others have been set up to test performance under 
different video modes. When you have selected the .ini file you want, click on 
the OK button to shut down GEOS and restart using the new .ini file.

Note that your personal .ini file is preserved-its [paths] *ini* field is updated 
to link in your selected platform .ini file.

If you wish to set up your own platform .ini file, make sure that it is in the 
proper subdirectory and contains a description of what it is simulating. 
Debug looks for .ini files in the INI subdirectory of the top-level GEOS 
directory. The platform descriptions are stored in the notes field of the 
[system] category in the .ini file. See one of the provided .ini files for an 
example of this.

### 6.2 Switching Kernels

The "Profile" item of the Options menu allows you to switch between the 
regular and profiling GEOS kernels. The profiling kernel runs slower than the 
regular kernel, but compiles information useful for optimizing geodes. 

When debugging, you may have several kernels residing in your SYSTEM 
directory. Debug creates a batch file which copies the appropriate kernel to 
geos.geo (or geosec.geo on an error-checking system). This kernel will then be 
used when GEOS restarts.

[Tool Command Language](ttcl.md) <-- &nbsp;&nbsp; [Table of Contents](../tools.md) &nbsp;&nbsp; --> [Icon Editor](ticoned.md)