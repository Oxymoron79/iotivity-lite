Java Language binding for IoTivity-Lite using SWIG
=================================================

Introduction
-------------------------------------------------
A tool called SWIG is used to generate Java language bindings for IoTivity-Lite.  SWIG is an
interface compiler that connects programs written in C and C++ with other languages such as Java.

SWIG is not a stubs generator.  It produces code that can be compiled and used.  The output is
generated by a tool.  For this reason there is little or no encapsulation of the code to make it
appear more like what most Java developers would expect.  The output with very few
exceptions are just a collection of classes filled with static methods.  This has both the
advantage and disadvantage that the way the code is used in Java is similar to the way it is used
in C.  This means the code does not resemble what a typical Java programmer expects but it is easy
for someone familiar with the C code to follow the flow of the code.

Since the Java output is just a thin layer on top of IoTivity-lite C code any programs developed
using Java are expected to be as stable as the underlying C code.

Getting Started
-------------------------------------------------
To use this code you will need the following:
  - git version control system
  - A local copy of IoTivity-lite
  - SWIG installed on your local system (version 3.0 recommended)
  - Java Development kit.
  - C build tool for your desired system

The SWIG output should work with Java 6 and newer.  The Output has been tested against OpenJDK 1.8.
Due to licensing changes the output is no longer tested against Oracle Java.

Currently SWIG version 3.0 is recommended.  SWIG version 4.0 came out during the  development of the
Java language bindings.  Users are welcome to use a newer version as long as they are aware it has
not been as extensively tested.

Instructions for Android
-------------------------------------------------
Instructions for Android can be found <iotivity-lite>/port/android/README.md

Instructions for Linux
-------------------------------------------------
### Install build tools
- git, swig, Java Development kit, make, and C compiler

    sudo apt-get install git swig build-essential openjdk-8-jdk make

- A local copy of IoTivity-lite

Checkout IoTivity-Lite git project run the following command to get a anonymous copy of
iotivity-lite.

    git clone https://gitlab.iotivity.org/iotivity/iotivity-lite.git

If you are behind a proxy setup the git proxy settings before running the above commands

    git config --global http.proxy http://<username>:<password>@<proxy-server-url>:<port>

Since this is an anonymous download you will not be able to push any changes up to the project.
If you are planning on contributing back to IoTivity you will need to make a gitlab account and
request developer access to the iotivity-lite project.

### Build Java language bindings
Navigate to `<iotivity-lite>/port/linux`

    make IPV4=1 DEBUG=1 JAVA=1 IDD=1

If make fails check see the Verify installation of needed tools section.

### Building and Running Samples
A sample server and client can be found in `<iotivity-lite>/swig/apps/<sample>`

The server sample is in `java_lite_simple_server`.  To build and run the sample execute the
following commands.

    ./build-simple-server-lite.sh
    ./run-simple-server-lite.sh

The client sample is in `java_lite_simple_client`.  To build and run the sample execute the
following commands.

    ./build-simple-client-lite.sh
    ./run-simple-client-lite.sh

The Onboarding tool sample is in `java_onboarding_tool`. To build and run the onboarding tool
execute the following commands.

    ./build-onboarding-tool-lite.sh
    ./run-onboarding-tool-lite.sh


See the Simple Step-by-Step guide for onboarding and provisioning section found in the root level
README for step-by-step instructions to onboard and test the samples.

Instructions for Windows
-------------------------------------------------
### Install build tools
 - Git can be obtained for Windows [here](https://git-scm.com/download/win).
 - SWIG can be downloaded [here](http://swig.org/download.html).
 - Java Development Kit (JDK) choose one (AdoptOpenJDK recommended)<br />
   AdoptOpenJDK can be [downloaded here](https://adoptopenjdk.net/installation.html)<br />
   Oracle Java 8 JDK can be
   [downloaded here](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html).
 - Download Visual Studio [here](https://visualstudio.microsoft.com/).

   For development on Windows Visual Studio 2015 was used.  The Visual Studio solution files have
   been tested in newer versions of Visual Studio and have been found to work.  Visual studio IDE
   Community edition and Visual Studio Code should work but have not personally been tested.

Set JAVA_HOME environment variable to point to the Java Development kit. This is required so the
build can find the jni.h header files.

### Build Java language bindings
Build the JNI shared library:

Navigate to `<iotivity-lite>/port/windows/vs2015` open the Visual Studio solution
`IoTivity-lite-Java.sln` file in Visual Studio.

Select the desired build options: Release/Debug, x86/x64.  Note the x86/x64 must match the
architecture of the Java VM installed on the system.  In the Solution Explorer right click
on the `iotivity-lite-jni` project.  Select the `Build` option.

This will build
  - `IoTivity-lite.lib`
  - swig generated wraper code
  - `iotivity-lite-jni.dll`

On success the Output window should show:

    ========== Build: 3 succeeded, 0 failed, 0 up-to-date, 0 skipped ==========

Build `iotivity-lite.jar` file from `<iotivity-lite>/swig/java_lang` run:

    sh build-iotivity-lite.sh

The Java build has not been integrated with Visual Studio. Running build-iotivity-lite.sh script is
a required extra step in addition to the Visual Studio build.

If the `iotivity-lite-java` project is imported into Eclipse then the Eclipse IDE can be used to
build `iotivity-lite.jar` instead of the shell script. See "Eclipse project files" section below.

If the build fails see the "Verify installation of needed tools" section.

### Building and Running Samples
A sample server and client can be found in `<iotivity-lite>/swig/apps/<sample>`

The server sample is in `java_lite_simple_server`.  To build and run the sample execute the
following commands.

    sh build-simple-server-lite.sh
    run-simple-server-lite.cmd

The client sample is in `java_lite_simple_client`.  To build and run the sample execute the
following commands.

    sh build-simple-client-lite.sh
    run-simple-client-lite.cmd

The Onboarding tool sample is in `java_onboarding_tool`. To build and run the onboarding tool
execute the following commands.

    sh build-onboarding-tool-lite.sh
    run-onboarding-tool-lite.cmd

See the Simple Step-by-Step guide for onboarding and provisioning section found in the root level
README for step-by-step instructions to onboard and test the samples.

### Windows specific build issues
Visual Studio does not know how to properly rebuild when the swig interface files have been change
(i.e. *.i and *.swg files found in the swig_interfaces directory) if any of these files are updated.
The **Rebuild** option must be used to build the output or the swig output will not be updated or
`sh build_swig.sh` can be run from the ``<iotivity-lite>/swig/java_lang` directory and then run
**Build**.

Visual Studio does not clean the *.java output files even on a clean or rebuild.  The files may
manually be deleted from the `<iotivity-lite>/swig/iotivity-lite-java/src/org` directory or
`sh build_swig.sh` can be run from the ``<iotivity-lite>/swig/java_lang` directory.

Layout of swig directory
-------------------------------------------------
This contains an overview of the contents of the `<iotivity-lite>/swig` directory.  With a
summary of the contents of each directory.

    swig
    +-- apps
    |   +-- android_simple_client
    |   +-- android_simple_server
    |   +-- java_lite_simple_client
    |   |   +-- src
    |   +-- java_lite_simple_server
    |   |   +-- src
    |   +-- oc
    +-- iotivity-lite-java
    |   +-- junit
    |   +-- src
    +-- java_lang
    +-- oc_java
    |   +-- oc
    +-- swig_interfaces

- `apps`<br />
  Contains multiple samples.  The `java_lite` samples have been run and tested on Windows and Linux.
  The `android` samples are the same samples with a really light UI for Android OS. The `java_lite`
  samples also contain project files for the Eclipse IDE if users wish to import the samples into
  that IDE. The `java_onboarding_tool` is a bear bones tool for onboarding and provisioning samples
  that have been built with security.

  The `oc` directory is a collection of samples that take advantage of the org.iotivity.oc package
  which is a layer built on top of the output generated by swig that gives the code a more object
  oriented feel and usage.
- `iotivity-lite-java`<br />
  Contains unit test code in the `junit` directory as well as an empty `src` directory.  The
  `src` directory will be populated with `*.java` files when the SWIG build commands are run.
  This also contains project files for the Eclipse IDE if user wishes to import the
  iotivity-lite code into Eclipse.
- `java_lang`<br />
  Contains build scripts that may be used to generate the Java language binding using SWIG.  Most of
  the scripts have been incorporated with make or Visual Studio and no longer need to be called.
- `oc_java`<br />
  Contains Java files that are used by the SWIG output but not not generated as part of the SWIG
  build process.  Most of the files are Java interfaces used to handle callbacks and bitmask
  values.

  The `oc` directory contains a layer ontop of the swig generated output that is more object oriented
  than the output from swig.
- `swig_interfaces`<br />
  Contains the input files for the swig builder.  These files contain instructions for the SWIG
  builder.  They tell swig which header files are being processed.  It instructs swig how to rename
  files from a C style name with underscores to a Java like lower camel case name.  It also
  instructs swig how to work with data types that it does not understand by default.  Data types
  like `oc_string_t`.  It also contains code that works around the fact that Java does not have a
  preprocessor so must handle many of the C macros differently.

Eclipse project files
-------------------------------------------------
Where possible command line build scripts have been provided for building and running the code.
However much of the development was done using the Eclipse IDE.  The project files have been
committed so other developers can take advantage of the Eclipse IDE if they wish.

To open the project files:
 - Open Eclipse
 - select `File->Import..`
 - select `General->Existing Projects into Workspace`
 - click `Next>` button
 - click the `Browse..` button next to the `Select root directory:` text box
 - browse to the `<iotivity-lite>\swig` directory click `OK`
 - make sure the checkbox for the desired projects is checked.  Click `Finish`

Run the unit tests by right clicking on `iotivity-lite` project select `Run As -> JUnit Test`.
Selecting `Run As -> Java Application` will run the command line version of the unit tests.

Run samples by right clicking on the sample select
`Run As -> Java Application`.

If the code was previously built using the command-line build scripts you may get build warnings
indicating some class files can not be found.  Select `Project -> Clean...`.  Then right click on
the `iotivity-lite` project and select `Refresh`.  This should force the project to rebuild the
all the class files associated with the `iotivity-lite.jar` file.

Verify installation of needed tools
-------------------------------------------------
If issues are encountered when trying to build the code verify the tools can be found. The scripts
assume all the needed tools are on PATH and are accessible without knowing the location of the tool.

Run the following to verify each tool.  At this time there are no known issues limiting users to
the same version of the tools as were used for development.  Feel free to use the latest version of
all the development tools.

---
bash shell (windows only) this should be installed with git

    sh --version

example of expected output

    GNU bash, version 4.4.12(2)-release (x86_64-pc-msys)
    Copyright (C) 2016 Free Software Foundation, Inc.
    License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>

---
git

    git --version

example of expected output (newest version recommended)

    git version 2.20.1

---
SWIG

    swig -version

example of expected output (version 3.0 currently recommended)

    SWIG Version 3.0.12
    Compiled with g++ [x86_64-redhat-linux-gnu]
    Configured options: +pcre
    Please see http://www.swig.org for reporting bugs and further information

---
Java

    java -version
    javac -version

example of expected output

    openjdk version "1.8.0_191"
    OpenJDK RuntimeEnvironment (build 1.8.0_191-b12)
    OprnJDK 64-bit Server VM (build 25.191-b12, mixed mode)

    javac 1.8.0_191

---

Check JAVA_HOME environment variable
(Windows)

    echo %JAVA_HOME%

example of expected output

    C:\Program Files\AdoptOpenJDK\jdk-8.0.202.08

(Linux)

    echo $JAVA_HOME

example of expected output

    /usr/lib/jvm/java-1.8.0/

---
If any tools are not found make sure the location of the tool is added to the system PATH.

If JAVA_HOME is not found add it to your environment variables.

Send Feedback
-------------------------------------------------
Questions
[IoTivity-Lite Developer Mailing List](https://iotivity.groups.io/g/iotivity-dev)

Bugs
[IoTivity-lite gitlab issues](https://gitlab.iotivity.org/iotivity/iotivity-lite/issues)