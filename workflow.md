# Running SIR Programs to Collect Fault-revealing and Coverage Data

[SIR](http://sir.unl.edu/portal/index.php) (Software-artifact Infrastructure Repository) is a collection of software programs, test cases, and faults for controlled experiments of software testing or program analysis research. A typical usage of SIR is to run the subject programs with accompanying test cases and (artificial) faults, and then collect the coverage and fault-revealing information of the test cases for further study. This note briefly introduces  this workflow, with an example of `make`, a GNU tool controlling compile and build process. 

## Download data

While we follow the workflow of SIR, the subject programs coming with SIR are reported too old to compile and run correctly [1]. As a result, we are using the artifacts organized by Henard et al. [1]. The subject programs are basically the same as the SIR ones, but with configuration scripts that make the compile and build easier in recent OS and machines. They also come with more mutants (faults) than the original SIR programs.

To run the following example, download the subject programs, regression test suites, and mutants from here:

https://henard.net/research/regression/ICSE_2016/

Unzip the files and put them together under a folder (I use `SIR-note` here). We are using `make_v5` in our example, so make sure you have something in these places:

```
subject_programs/make/v5
test_suites/make
mutants/make_v5_mutants/
```

To run the test cases of `make`, we also need the test scripts of `make` in the original SIR folder. Download `make_1.4.tar.gz` from here:

http://sir.unl.edu/php/showfiles.php (free registration required)

Untar the file, and make sure there is something under `make/testplans.alt/testscripts/`

Finally, to follow the SIR workflow, we have to create some folders. Let's create a new folder called `exp` first, and the following subfolders:

```
castman@castman-vm-mint ~/SIR-note/exp $ tree -d
.
├── inputs
├── inputs.alt
├── outputs
├── outputs.alt
├── scripts
├── source
└── testplans.alt
    └── testscripts
```

That's it. We are ready to go!

## The workflow

In general, the workflow to collect fault-revealing information of test cases for an SIR subject program is as follows:

1. Copy original version of the program to `source`.
2. Configure and build original code in `source`, usually by `./configure; make`, or `make build`. (Some environment variables and compiler options may need to be set to build the SIR C programs. Refer to [here](http://sir.unl.edu/content/c-overall.php#Overview) and [here](http://sir.unl.edu/content/configuration-support.php))
3. Generate a test script (usually by [the MTS tool](http://sir.unl.edu/content/mts-usage.php)) to execute test suite against original version, and save the results to `outputs` for future reference.
4. Move the oracle outputs from `outputs` to `outputs.alt`. Clear `source`.
5. Copy original version of the program to `source` again, and then replace the files in `source` with a mutant (one or more faulty files). 
6. Repeat step 2 to build the faulty version of the program in `source`.
7. Repeat step 3 to execute test suite against the faulty version and save the results to `outputs`
8. Outputs for the same test cases against the original version and the mutant are in `outputs` and `outputs.alt`, respectively. Compare them to know if the mutant got killed by a test case. If you are using the MTS tool, the comparison is automatically done by the test script.

In the original SIR workflow, original versions of programs are in `versions.alt/versions.orig/vK`, and mutants are in `versions.alt/versions.seeded/vK`. Moreover, different mutants may be triggered by modifying `source/FaultSeeds.h`. Here we are using subject programs and mutants from [1], so the locations of original versions and mutants are different in the following example.

## A running example with `make_v5`

We starts from the `exp` directory
```
castman@castman-vm-mint ~/Desktop/SIR-note/exp $ pwd
/home/castman/Desktop/SIR-note/exp
```

Commands to run the above workflow are as follows:
```
# Clear data
rm -rf source/*
rm -rf outputs/*
rm -rf outputs.alt/*
rm -rf scripts/*
rm -rf inputs

# Copy data
# the original version of program
cp -r ../subject_programs/make/v5/* source/

# Test suites for make
cp -af ../test_suites/make/inputs .

# Additional test scripts for make
cp -af ../make/testplans.alt/testscripts/* testplans.alt/testscripts/

# An executable, rm-makestuff, is needed to execute the generated test script.
# Rebuild to make it working on our OS/machine
(cd testplans.alt/testscripts; gcc rm-makestuff.c -o rm-makestuff)

# Build the original version
(cd source; ./configure CFLAGS="-g"; make)

# Generate a test script to execute test suite
# Refer to http://sir.unl.edu/content/mts-usage.php
# to install and configure the MTS tool 
(cd scripts; mts .. ../source/make ../../test_suites/make/make.tests R test.sh NULL NULL)

# test.sh was generated. Adjust some paths in the file
# to accommodate our current folder structure
(cd scripts; sed -i 's/..\/source/..\/..\/source/g' test.sh; sed -i 's/> ..\/outputs/> ..\/..\/outputs/g' test.sh)
(cd scripts; sed -i 's/hello ..\/outputs\//hello ..\/..\/outputs\//g' test.sh)

# Run the test cases
(cd scripts; ./test.sh)

>>>>>>>>running test 1
>>>>>>>>running test 2
>>>>>>>>running test 3
...
>>>>>>>>running test 109
>>>>>>>>running test 110
>>>>>>>>running test 111

# Move oracle outputs to outputs.alt
mv outputs/* outputs.alt

# Clear data and build the faulty version
rm -rf source/*
rm -rf outputs/*
rm -rf scripts/*
rm -rf inputs

# Original version and test suite
cp -r ../subject_programs/make/v5/* source/
cp -af ../test_suites/make/inputs .

# Mutant #18985
cp ../mutants/make_v5_mutants/18985/* source/

# Build the faulty version
(cd source; ./configure CFLAGS="-g"; make)

# Generate a test script to execute test suite
# Note that here the parameters for the MTS tools are different
(cd scripts; mts .. ../source/make ../../test_suites/make/make.tests D test.sh NULL ../outputs.alt)

# test.sh was generated. Adjust some paths in the file
# to accommodate our current folder structure
(cd scripts; sed -i 's/..\/source/..\/..\/source/g' test.sh; sed -i 's/> ..\/outputs\//> ..\/..\/outputs\//g' test.sh)
(cd scripts; sed -i 's/hello ..\/outputs\//hello ..\/..\/outputs\//g' test.sh)

# Run the test cases
(cd scripts; ./test.sh)

>>>>>>>>running test 1
>>>>>>>>running test 2
>>>>>>>>running test 3
...
>>>>>>>>running test 31
different results
>>>>>>>>running test 32
different results
>>>>>>>>running test 33
different results
different results
...
```
In the second run, `test.sh` runs each test case and compares the results under `outputs` and `outputs.alt`. If the results are different, `different results` is printed, indicating that the mutant was killed by the test case. Let's check the results of test 33. From `test.sh` we can see one of the output files by test 33 is `t92.out`:

```
castman@castman-vm-mint ~/Desktop/SIR-note/exp $ diff outputs/t92.out outputs.alt/t92.out 
11a12,15
> gcc -c  -DSTDC_HEADERS=1 -DHAVE_STRING_H=1 -DHAVE_FCNTL_H=1 -DHAVE_SYS_FILE_H=1 -DHAVE_ALLOCA_H=1 -g version.c
> gcc -c  -DSTDC_HEADERS=1 -DHAVE_STRING_H=1 -DHAVE_FCNTL_H=1 -DHAVE_SYS_FILE_H=1 -DHAVE_ALLOCA_H=1 -g getopt.c
> gcc -c  -DSTDC_HEADERS=1 -DHAVE_STRING_H=1 -DHAVE_FCNTL_H=1 -DHAVE_SYS_FILE_H=1 -DHAVE_ALLOCA_H=1 -g getopt1.c
> gcc -g -o hello hello.o version.o getopt.o getopt1.o  
```

The results of the same test case (t33) are different, so t33 killed the mutant. However, if we check the results (`t88.out`) of test 31:

```
@castman-vm-mint ~/Desktop/SIR-note/exp $ diff outputs/t88.out outputs.alt/t88.out 
13c13
< # Make data base, printed on Thu Apr 26 23:46:08 2018
---
> # Make data base, printed on Thu Apr 26 23:45:53 2018
1377c1377
< # Finished Make data base on Thu Apr 26 23:46:08 2018
---
> # Finished Make data base on Thu Apr 26 23:45:53 2018
```

The only difference is the timestamps. Should we say test 31 kills the mutant? I don't think so. From this example, you can see that we may have to further check the results reported by the MTS tool.

## Collect coverage information

We can also collect coverage information of test cases by following the first three steps of the workflow. Here we are using [GNU Gcov](https://gcc.gnu.org/onlinedocs/gcc/Gcov.html) to collect coverage information.

```
# Clear data
rm -rf source/*
rm -rf outputs/*
rm -rf outputs.alt/*
rm -rf scripts/*
rm -rf inputs

# Copy data
cp -r ../subject_programs/make/v5/* source/
cp -af ../test_suites/make/inputs .
cp -af ../make/testplans.alt/testscripts/* testplans.alt/testscripts/
(cd testplans.alt/testscripts; gcc rm-makestuff.c -o rm-makestuff)

# Build. Note that here we add some parameters to use Gcov
(cd source; ./configure CFLAGS="-g -fprofile-arcs -ftest-coverage"; make)

# Generate a test script to execute test suite
(cd scripts; mts .. ../source/make ../../test_suites/make/make.tests R test.sh NULL NULL)
(cd scripts; sed -i 's/..\/source/..\/..\/source/g' test.sh; sed -i 's/> ..\/outputs/> ..\/..\/outputs/g' test.sh)
(cd scripts; sed -i 's/hello ..\/outputs\//hello ..\/..\/outputs\//g' test.sh)

# Move test.sh to source/ for Gcov, and run
(mv scripts/test.sh source/)
(cd source; ./test.sh)

# Generate coverage files
(cd source; gcov -b *.c)
```

You can see the coverage information is generated as `*.gcov`. This is the coverage information for the whole test suite. To get the information for each test case, a possible way is to modify the generated `test.sh`.

[1] Christopher Henard, Mike Papadakis, Mark Harman, Yue Jia, and Yves Le Traon. Comparing white-box and black-box test prioritization. ICSE'16.
