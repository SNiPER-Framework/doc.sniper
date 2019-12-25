# Algorithms

The algorithm is the basic block of a data processing procedure. In this chapter, we will show how to implement and use an algorithm via a very simple example. The content covers the C++ algorithm code and Python configuration. Please see the appendix to find how to setup the environment and compile a package by CMT.


## The C++ Code

Suppose we have created a package named FirstAlg, and we want to define an algorithm with the same name. The necessary work includes,

- define a C++ class named FirstAlg, which is inherited from the AlgBase class;
- define a constructor, which takes a string parameter; implement the 3 abstract methods in base class, initialize(), execute() and finalize(); The bool return value indicates the execution status. If there is any error, the return value should be `false`, and the application will be stopped.

### Class defination

In the example, we will also involve the properties, which are defined as the data members of the algorithm class.

The contents of the header file (FirstAlg.h),

```c++
// file FirstAlg.h
#ifndef FIRST_ALG_H
#define FIRST_ALG_H

#include "SniperKernel/AlgBase.h"  //the AlgBase defination

class FirstAlg : public AlgBase
{
    public:
        FirstAlg(const std::string& name);  //constructor

        bool initialize();
        bool execute();
        bool finalize();

    private:
        unsigned int  m_count;  //the count of event loop
        int           m_value;  //an int property
        std::string   m_msg;    //a string property
};
#endif
```

### Algorithm declaration

An algorithm must be declared in SNiPER, so that it can be registered into the framework when the module is loaded. This is done by a macro DECLARE_ALGORITHM, which takes the algorithm class name as its parameter. It's recommended to put the declaration at the beginning of the source file.

```c++
DECLARE_ALGORITHM(FirstAlg);
```

### Properties

Properties are the C++ variables that configurable in Python at run time. In an algorithm, a property can be declared by the ```declProp()``` method. This method takes 2 parameters, the first is the name of the property, and the second is its correlated variable.

The following variable types can be declared as properties,

- scalar: C++ build in types and std::string;
- std::vector with scalar element type
- std::map with scalar key type and scalar value type

The properties have to be declared in the constructor. We can set a default value while the declaration. For example,

```
declProp("TheValue", m_value = 1);
```

### Logs

We can print any message via ```SniperLog``` just like the ```std::cout```.

```
LogDebug << "in the FirstAlg::execute()" << std::endl;
```

The corresponding message on the screen is,

```
task:FirstAlg.execute         DEBUG: in the FirstAlg::execute()
```

At the beginning, it shows this is a log printed in the execute() method of FirstAlg. Then, it is an indicator of the log level. And finally it is the log content.

There are 6 log levels in SNiPER. From lowest to highest they are,

- 0: LogTest, the indicator is "TEST";
- 2: LogDebug, the indicator is "DEBUG";
- 3: LogInfo, the indicator is "INFO";
- 4: LogWarn, the indicator is "WARN", means this is a warning message;
- 5: LogError, the indicator is "ERROR";
- 6: LogFatal, the indicator is "FATAL";

We can set the log level globally via Task, or set a different log level to each DLE component. Then the logs with lower levels will not shown on the screen.

### The implementation

We put the contents together. Here is the implementation of the source file (FirstAlg.cc),

```c++
// file FirstAlg.cc
#include "FirstAlg.h"
#include "SniperKernel/AlgFactory.h"  //the macro DECLARE_ALGORITHM

DECLARE_ALGORITHM(FirstAlg);

FirstAlg::FirstAlg(const std::string& name)
    : AlgBase(name),
      m_count(0)
{
    declProp("TheValue", m_value = 1);
    declProp("Message",  m_msg);
}

bool FirstAlg::initialize()
{
    LogDebug << "in the FirstAlg::initialize()" << std::endl;
    return true;
}

bool FirstAlg::execute()
{
    LogDebug << "in the FirstAlg::execute()" << std::endl;

    ++m_count;
    m_value *= m_count;
    LogInfo << "Loop " << m_count << m_msg << m_value << std::endl;

    return true;
}

bool FirstAlg::finalize()
{
    LogDebug << "in the FirstAlg::finalize()" << std::endl;
    return true;
}
```


## Python Configuration and Execution

### Load and Register an Algorithm

After the compilation, a dynamic library is generated. Suppose it is named as "libFirstAlg.so". We can load the library in python like this,

```python
import Sniper
Sniper.loadDll("libFirstAlg.so")
```

Then the algorithm FirstAlg is automatically registered and can be used in SNiPER.

However, if we want to use the package in Python style, we can wrap the previous code in a Python module. In the FirstAlg package, we can make the Python module by adding the code in the following file (please keep the directory structure),

```
python/FirstAlg/__init__.py
```

Then we can load and register FirstAlg with a single line in the main Python configuration script,

```python
import FirstAlg
```

A SNiPER application is always start from a Task instance. When an algorithm is registered, we can create its instance via the Task,

```
task.createAlg("FirstAlg")
```

### Configure a Job

The following Python code shows a complete executable SNiPER application,

```python
#!/usr/bin/env python

import Sniper
task = Sniper.Task("task")  # create a Task instance
task.setEvtMax(3)  # events loop number (3 times)
task.setLogLevel(2)  # the SniperLog print level

import FirstAlg
alg = task.createAlg("FirstAlg")  # create a FirstAlg instance
alg.property("TheValue").set(2)
alg.property("Message").set(" the value is ")

task.run()
```

In the example, we will execute the FirstAlg 3 time during events loop. The 2 properties are set via their names. The values of the correlated C++ variables will be directly modified. The result of this configuration is,

```
$ python run.py
*****************************************
***     Welcome to SNiPER Python      ***
*****************************************
Running @ lxslc603.ihep.ac.cn on Mon Dec 10 10:18:39 2017

task:FirstAlg.initialize      DEBUG: in the FirstAlg::initialize()
task.initialize                INFO: initialized
task:FirstAlg.execute         DEBUG: in the FirstAlg::execute()
task:FirstAlg.execute          INFO: Loop 1 the value is 2
task:FirstAlg.execute         DEBUG: in the FirstAlg::execute()
task:FirstAlg.execute          INFO: Loop 2 the value is 4
task:FirstAlg.execute         DEBUG: in the FirstAlg::execute()
task:FirstAlg.execute          INFO: Loop 3 the value is 12
task:FirstAlg.finalize        DEBUG: in the FirstAlg::finalize()
task.finalize                  INFO: finalized

***  SNiPER Terminated Successfully!  ***
```

### DLE Instance Name

In the previous section, we defined a constructor with a string parameter in FirstAlg. That means we can assign a name to the algorithm instance. In fact, each DLE (algorithm, service and task) instance has a name, and the name can be customizd. The default name of algorithm/service is its class name. We can assign a different one when we create the instance,

```
task.createAlg("FirstAlg/NewName")
```

The sub-string after "/" is the instance name we assigned.

In this way, we can create more than one instances of an algorithm without conflicts. For example,

```python
#!/usr/bin/env python

import Sniper
task = Sniper.Task("App")
task.setEvtMax(3)
task.setLogLevel(3)

import FirstAlg
alg1 = task.createAlg("FirstAlg/Alg1")
alg1.property("TheValue").set(2)
alg1.property("Message").set(" the value is ")

alg2 = task.createAlg("FirstAlg/Alg2")
alg2.property("TheValue").set(20)
alg2.property("Message").set(" the value is ")

task.run()
```

In this example, we create 2 instances of FirstAlg with different names Alg1 and Alg2. Please notice that,

- we assign a new name ("App") to the Task instance, too;
- we changed the log level to 3, so the DEBUG messages will not be printed;
- the 2 algorithm instances, Alg1 and Alg2, have different property values;

Then the result of this configuration is,

```
$ python run.py
*****************************************
***     Welcome to SNiPER Python      ***
*****************************************
Running @ lxslc603.ihep.ac.cn on Mon Dec 10 10:56:17 2017

App.initialize                 INFO: initialized
App:Alg1.execute               INFO: Loop 1 the value is 2
App:Alg2.execute               INFO: Loop 1 the value is 20
App:Alg1.execute               INFO: Loop 2 the value is 4
App:Alg2.execute               INFO: Loop 2 the value is 40
App:Alg1.execute               INFO: Loop 3 the value is 12
App:Alg2.execute               INFO: Loop 3 the value is 120
App.finalize                   INFO: finalized

***  SNiPER Terminated Successfully!  ***
```

Please compare the logs and find the difference to the previous example. It is helpful to understand the DLE names and the SniperLog.
