# README #

Interface to run AFL on Java programs.

Kelinci means rabbit in Indonesian (the language spoken on Java). 

This README assumes that AFL has been previously installed. For information on how to install and use AFL, please see <http://lcamtuf.coredump.cx/afl/>. Kelinci has been tested successfully with AFL versions 2.44 and newer. The README explains how to use the tool. For technical background, please see the CCS'17 paper in the 'docs' directory. 

### Set up ###

The application has two components. First, there is a C application that acts as the target application for AFL.
It behaves the same as an application built with afl-gcc / afl-g++; AFL cannot tell the difference.
This C application is found in the subdirectory 'fuzzerside'. It sends the input files generated by AFL
to the JAVA side over a TCP connection. It then receives the results and forwards them to AFL in its
expected format. To build, run `make` in the 'fuzzerside' subdirectory.

The second component is on the JAVA side. It is found in the 'instrumentor' subdirectory.
This component instruments a target application with AFL style administration, plus a component to communicate
with the C side. When later executing the instrumented program, this sets up a TCP server and runs the target 
application in a separate thread for each incoming request. It sends back an exit code (succes, timeout, crash 
or queue full), plus the gathered path information. Any exception escaping main is considered a crash.  
To build, run `gradle build` in the 'instrumentor' subdirectory. Next, run the Instrumentor on a class or JAR with
`java -jar build/libs/kelinci.jar -i <class-or-jar> -o <output-dir>` (you'll need to make sure that the input class
 is also on the classpath). This should print which classes were written to the output directory. Next, we want to 
start the server that will handle requests from the fuzzer side and execute the target application on them. 
Run `java -cp <output-dir> edu.cmu.sv.kelinci.Kelinci [-v V] [-p P] [-t T] target.Application <target-args>`, 
where <output-dir> is the directory where the Instrumentor wrote its output, V is an optional verbosity 
level (0-3 for silent to very verbose), P is an optional port number (default is 7007), T is an optional 
time-out in milliseconds (default is 300000 / 5 minutes) and <target-args> are the usual parameters to the target
application. Kelinci will pass the provided arguments on to the target application. At least one of the arguments
should be @@, which will be replaced by the path to the file on which it will be executed.

Once the server is running, we can start AFL. We will need a directory with at least one input file from
which to start fuzzing, e.g. 'in\_dir'. Let's start by checking that the connection with the server is running,
by running the interface without AFL: `./interface in\_dir/input.txt` (or similar, for some input file).
The last line output by this test should be 'Received results. Terminating.'. If the connection is working, we
can run AFL with `afl-fuzz -i in\_dir -o out\_dir ./interface @@`. This assumes that AFL has been compiled
on the system and an alias for 'afl-fuzz' has been set-up.

In case one wishes to run multiple instances of the Java side in parallel, one may use the -s flag for interface.c to specify a file that lists the servers to use. Each line in the file lists the server (URI or IP) and optionally the port after a colon. An example servers file is:
```
localhost:7007
1.1.1.1:7008
2.2.2.2
3.3.3.3:7009
```
The Java side application does not handle multiple input files in parallel, as those runs could potentially interfere with one another. Instead, if one wants to leverage multiple processor cores on one systems, multiple instances of the Java side application should be started listening to different ports.

### Developer ###

Rody Kersten (rodykersten@gmail.com)
