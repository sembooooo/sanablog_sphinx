# clean coding applied to Testing
## Introduction
People who are reading this are my motivation to write this article. I wanted to write 
one article like this for so long but i never was motivated and now i am . So boy! Lets go.
Please feel free to critique my writing, dont hesitate to even abuse. Abuses with proper explanation are taken not only to the mind but to the heart.
srikarsana1995@gmail.com is my email id.

## Clean coding
I have not encountered a Clean coding book which emphasized on testing. I only read two books but they only contain one chapter maximum dedicated to this subject. The biggest learning till now in my developement career is   
               **"Your code is not clean until your testcases are clean"**.

## Brief info about the software module
The software module that is considered here is inspired from my previous experience. Lets call it power supply module for an [Electronic Control Unit(ECU)](!https://en.wikipedia.org/wiki/Electronic_control_unit).
This power supply module does few things

1. As soon as the ECU starts the software module should test the Supply paths hardware. There are different tests which are conducted on the paths to determine its health. For simplicity lets name Test_1 ,Test_2 and Test_3. Talk about bad naming 🤣 .
1. There are multiple power supply connections to the ECU. After the supply paths startup tests are completed, this software module looks at the power supply voltages from different external sources and selects the best one based on the available voltage. So it will switch On the transistors/MOSFETs in the supply path in a fashion so that the ECU is completely supplied by that one specific external power supply.  We call this a switch over.
1. After all the startup tests of the ECU are done then algorithm software is executed. At times Algorithm  also commands us , based on the usecase, to switch over to a different power supply source that is connected to the ECU. 
1. As this is an embedded system with an operating system , there are different processes which run at different frequencies like 10msec ,5msec ,25msec etc.,.
1. There will be different components in an ECU and our ECU has different supply paths too. Meaning other hardware components are not always supplied by a single power supply source but the source changes after a switchover. Whenever a supply path is changed other software components that are dependent on the source voltage should also now look at a different source. To implement this logic the other module owner should now learn about Supply paths which is not efficient. So the supply paths itself will try to provide  few voltage interfaces( called wrapper voltage interface ) which hides the implementation details and only provides the voltage that each module is being supplied with.
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
Every unit has inputs and outputs. One should be able to invoke all the code written in the unit just by using inputs and outputs. I see benefits by not accessing the implementation details.

### Statemachine 
There are few statemachines (which are implemented using a switch case statements) in this module.
I see developers straight away setting all the global variables needed and the state variable for the state machine too so that the testcase directly lands at a particular state.
Doing so is harmful as we might not calculate every value correctly every single time. So let the program calculate it for you. Lets say there are 10 states in the statemachine and one need to evaluate the procedure happening at stage 6th. One can run the statemachine process from first state to 6th state
but instead developers chose to manipulate the global variables that control the statemachine.
When asked why resort to such lazy implementation in testcases the developers answered me that they felt UT execution might be slowed down because of the for loop. I mean seriously on a 2.2GHz system how can one feel that looping 6 times can slow down UT execution?

Consider the following testcase
```c
void Testcase_State6(void)
{
    Statemachine_t *State = get_StateRef(); // function only created to get state varaiable feference.
    TestControlData_t *TestControlData = get_TestControlDataRef();  // function only created to get test control data reference.
    *State = TestState_State6; 
    TestControlData->FailureStage = TestStage_NotValid;
    TestControlData->FailureCounter = 0;
    statemachine();
    EvalEq(isPathSwitchOFF(), TRUE);
}
void Testcase_State7(void)
{
    Statemachine_t *State = get_StateRef(); // function only created to get state varaiable feference.
    TestControlData_t *TestControlData = get_TestControlDataRef();  // function only created to get test control data reference.
    *State = TestState_State7; 
    TestControlData->FailureStage = TestStage_state6;
    TestControlData->FailureCounter = 4;
    statemachine();
    EvalEq(isFailureReported(), TRUE);
}
```
vs
```c
void Testcase_State6(void)
{
    setBatteryVoltage(12.0);
    for(int _5msecCyle =0;_5msecCyle <6;_5msecCyle++){
    statemachine();
    }
    EvalEq(isPathSwitchOFF(), TRUE);
}
void Testcase_State7(void)
{
    // setup
    setBatteryVoltage(12.0);
    for(int _5msecCyle =0;_5msecCyle <6;_5msecCyle++){
    statemachine();
    }
    setBatteryVoltage(0.0);
    // Execute 
    statemachine();
    // Evaluate 
    EvalEq(isFailureReported(), TRUE);
}
```

Which one is more readable and understandable ?
Benefits of not accessing implementation details

1. The testcase will be readable as there will be fewer lines of code. (mostly)
1. No need to modify testcases if we change the design of src code.
1. Less lines for the reviewer to go through.
1. Easy to identify problems as the lines of code is less. It is hard to identify mistakes in code which has more lines.
1. No need to write extra code just for testing.

### Testing the state machine in a single function
I have come across testcases that are of 1000 lines in size. Few Devs have their whole statemachine in a single testcase. 
A testcase should one test one thing and only one thing.

### Testing sub functions

lets assume the source code of `Supplypaths_switchover.c` as follows

```c

static void RouteProperVoltageToWrapperInterfaces(RRequest_t ReqeustedSupplyPath){
    /*some lines of code */
}
static void PerformSwitchOverTo(Request_t ReqeustedSupplyPath)
{
    /* some lines of code */
    /*
    Once we perform switchover 
    */
    RouteProperVoltageToWrapperInterfaces(ReqeustedSupplyPath);
}

void Process_switchover(void)
{
    if(AreAllStartupTestsPassed())
    {
        Request_t RequestedSupplyPath = get_PriorityRequest();
        PerformSwitchOverTo(RequestedSupplyPath); 
    }
    else
    {
        PerformSupplyPathShutdown();
        ReportFailureToSystem();
    }    
    
}
```
I have seen many people testing `RouteProperVoltageToWrapperInterfaces()` seperately. They write adapter/helper functions to invoke this static function alone but they never wrote the testcase using `Process_switchover()`. End of the day Code coverage is 100% .
In such a testing even if we comment the line as shown below the testcases will still pass. 
```c
static void RouteProperVoltageToWrapperInterfaces(RRequest_t ReqeustedSupplyPath){
    /*some lines of code */
}

void PerformSwitchOverTo(Request_t ReqeustedSupplyPath)
{
    /* some lines of code */
    /*
    Once we perform switchover 
    */
    //RouteProperVoltageToWrapperInterfaces(ReqeustedSupplyPath);
}

void Process_switchover(void)
{
    if(AreAllStartupTestsPassed())
    {
        Request_t RequestedSupplyPath = get_PriorityRequest();
        PerformSwitchOverTo(RequestedSupplyPath); 
    }
    else
    {
        PerformSupplyPathShutdown();
        ReportFailureToSystem();
    }    
    
}
```
Now we have missed a defect that could have been detected if `Process_switchover()` was used in the testcase that Evaluates wrapper voltage interface values for different requests.
Code coverage should be used a tool that helps us indicate the untested part of the code but one should not just rely on the code coverage tool alone to identify untested code part.
Never test subfunctions instead test it as a whole.

## Name of a testcase
People instead of naming the testcase they write so many comments.
I started using the naming convention from Roy Osherove book - The art of unit testing.
Test cases are named better if we follow the format `[Given]_When_Then`. 
In my experience file name explains `Given` and it becomes redundant to again mention it in every test case. So in those cases i resort to using `when_then` style.

Few examples of naming the testcases using the format which Roy explained are as follows

`SuccessfullStartupCompletion_AppSwCommandsStop_NoSwitchOverHappens()`
`UnSuccessfullStartupCompletion_NoCommandFromAppSw_NoSwitchOverHappens()`
`SuccessfullStartupCompletion_NoCommandFromAppSw_SwitchOverHappens()`

One thing which me and my reviwers experienced after following this style of naming was it was easy to identify missing testcases as this style helps us look the requirements from a different perspective. It was more like different combinations  of the words `SuccessfullStartupCompletion`, `AppSwCommands` and `witchOverHappening` .
Good naming also helped me with TDD.

## Duplication in testcode
People tend to ignore duplication in testcode. Eliminating duplication has a few good benefits.
1. Reviwer need not review the same code over and over again.
1. It takes less of code develop
1. All we have to do is change at one place and not every testcase.

Here is one Example of the duplication i have seen

```c
void Test1(void)
{
    /* some testcode */
    Eval(isFailureReportedToSystem());
    Eval(isSupplySwitchOverFailed());
    Eval(AreAllSupplyPathsSwitchedOFF());
}
void Test2(void)
{
    /* some testcode */
    Eval(isFailureReportedToSystem());
    Eval(isSupplySwitchOverFailed());
    Eval(AreAllSupplyPathsSwitchedOFF());
}
void Test3(void)
{
    /* some testcode */
    Eval(isFailureReportedToSystem());
    Eval(isSupplySwitchOverFailed());
    Eval(AreAllSupplyPathsSwitchedOFF());
}
```

Here there is duplication in Evaluation part which can be reduced. Every test case has three parts. Setup , execute and Evaluate . In my short experience as developer and reviwer i have only noticed duplication at the first (setup ) and the last Evaluate part.

## Why write stubs when you have the code at hand ?

This is also not my idea but i have picked it up from Roy's book. Why should one always treat 
a single file as a unit.
So in our case we have the following files
1. `Supplypaths_Test1.c`
1. `Supplypaths_Test2.c`
1. `Supplypaths_Test3.c`
1. `Supplypaths_Scheduler.c`
1. `Supplypaths_switchover.c`
1. `Supplypaths_internalRequestor.c`
1. `Supplypaths_ExternalRequestor.c`
1. `Supplypaths_RequestPrioritization.c`

Every file takes inputs from other files in the module. Few files take a lot of inputs and few other files take a very few inputs and few dont.
If we test every file seperately we have to write mocks/stubs for the interfaces that are internal to the component. Why cant we create a test environment which builds the whole module together. As it is only a few files , compilation time is not drastically increased.
Developement is ofcourse less as we do not need to write stubs for internal interfaces. 
Reviewer has less code to review.