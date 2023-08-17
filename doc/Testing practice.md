# My Learnings in Testing 
## Introduction
People who are reading this are my motivation to write this article. I wanted to write 
one article like this for so long but i never was motivated and now i am . So boy! Lets go.
Please feel free to critique my writing, dont hesitate to even abuse. Abuses with proper explanation are taken not only to the mind but to the heart.
srikarsana1995@gmail.com is my email id.

## Clean coding
I have not encountered a Clean coding book which emphasized on testing. I only read two books but they only contain one chapter maximum dedicated to this subject. The biggest learning till now in my developement career is   
               **"Your code is not clean until testcases are not clean"**.

```{note}
Short and crisp content ahead. No fillers
```
## Brief info about the software module
The software module that is considered is inspired from previous experience. Lets call it power supply module for an [Electronic Control Unit(ECU)](!https://en.wikipedia.org/wiki/Electronic_control_unit).
This power supply module does few things

1. As soon as the ECU starts the software module should test the Supply paths hardware. There are different tests which are conducted on the paths to determine its health. For simplicity lets name Test_1 ,Test_2 and Test_3. Talk about bad naming 🤣 .
1. There are multiple power supply connections to the ECU. After the supply paths startup tests are completed, this software module looks at the power supply voltages from the different external sources and chooses the best one. So it will switch On the transistors/MOSFETs in the supply path in a fashion so that the ECU is completely supplied by that one specific external power supply.  We call this switch over.
1. After all the startup tests of the ECU are done then algorithm software is executed. At times Algorithm  also commands us , based on the usecase, sto witch over to a different power supply source that is connected to the ECU. 
1. As this is an embedded system with an operating system , there are different processes which run at different frequencies like 10msec ,5msec ,25msec etc.,.
## Single Responsibility principle
### Bad Practice 
SRP is followed in the `src` folder. Different files are created based on the functionality for example.

1. `Supplypaths_Test1.c`
1. `Supplypaths_Test2.c`
1. `Supplypaths_Test3.c`
1. `Supplypaths_Scheduler.c`
1. `Supplypaths_switchover.c`
1. `Supplypaths_internalRequestor.c`
1. `Supplypaths_ExternalRequestor.c`
1. `Supplypaths_RequestPrioritization.c`

May be i have overmodularized it.  But the same is not followed in the unit testing. Developers tend to write 100's of testcases in a single file. Lets say if Test_1.c unit testing has 200 testcases then all these 200 testcases are written in a single file as `UT_Supplypaths_Test1.c`. 
### Good Practice
Instead why not break this UT file by following SRP principle as follows
1. `PreconditionsFailureTests.c`
1. `SwitchOFFTests.c`
1. `SwitchOnTests.c`

### Benefits

1. It will easier for your proxy or your successor to maintain the code
1. It will easier for the reviewer to review as the tests are modularized. May be this helps in giving out small meaningful PRs.
1. If you a mistake during your developement then the build log will let us know which part of the feature has bug as test cases from that specific file fails. Now the error log gives us more information without putting much work.
1. Lets say this happens on jenkins it will also be easy for integrators or other people to understand as the log is containing extra information.


## Using implementation details
```{note}
There are different meanings for a unit , for now i am considering a c file as a unit.
```
Every unit has inputs and outputs. One should be able to invoke all the code in the unit just by using inputs and outputs. One should not access the global variables and other things(not sure how to name it but examples on the way) while testing. I see benefits by not accessing them.



