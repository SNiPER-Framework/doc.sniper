# Services

A service is a subroutine that can be invoked by users on demand. In this chapter, we will show how to implement and use a service module. For the similarity between service and algorithm, we will concentrate on the differences, but not all the details.


## The C++ code

We will implement a service named FirstSvc.

For a service can be invoked by others, the header file should be visible globally. We can export the FirstSvc.h header file directly for simplicity. However, we can add a pure virtual class to declare the user APIs. So that we can implement different services with the same APIs. These services can be replaced with each other. This is an useful feature in a framework.

The defination of the pure virtual class is,

```c++
// file FirstSvc/IfMySvc.h
#ifndef IF_MY_SVC_H
#define IF_MY_SVC_H

class IfMySvc
{
    public:
        virtual void doSomeThing() = 0;
};
#endif
```

We put this header file in a seperated directory, and copy it into a directory that can be found by the compiler. (This is guaranteed by a pattern of CMT at present.)

We use a multiple inheritance in the defination of FirstSvc. The code is,

```c++
// file FirstSvc.h
#ifndef FIRST_SVC_H
#define FIRST_SVC_H

#include "FirstSvc/IfMySvc.h"
#include "SniperKernel/SvcBase.h"

class FirstSvc : public SvcBase, public IfMySvc
{
    public:
        FirstSvc(const std::string& name);

        bool initialize();
        bool finalize();

        void doSomeThing();
};
#endif
```

There is no `execute()` method. Instead, we should implement the method `doSomeThing()` as the user API.

The source file is,

```c++
// file FirstSvc.cc
#include "FirstSvc/FirstSvc.h"
#include "SniperKernel/SvcFactory.h"

DECLARE_SERVICE(FirstSvc);

FirstSvc::FirstSvc(const std::string& name)
    : SvcBase(name)
{
}

bool FirstSvc::initialize()
{
    LogDebug << "in FirstSvc::initialize()" << std::endl;
    return true;
}

bool FirstSvc::finalize()
{
    LogDebug << "in FirstSvc::finalize()" << std::endl;
    return true;
}

void FirstSvc::doSomeThing()
{
    LogInfo << "Do some thing in a service" << std::endl;
}
```

Like the algorithm, a service must be declared by a macro DECLARE_SERVICE. We can declare properties in the servcie constructor, too. But we will not repeat this here.

In this example, we only print a message in the `doSomeThing()` API.


## Invoke a Service

In this section we create a new algorithm, SecondAlg, in which the FirstSvc is invoked.

The header file is,

```c++
// file SecondAlg.h
#ifndef SECOND_ALG_H
#define SECOND_ALG_H

#include "SniperKernel/AlgBase.h"

class IfMySvc;

class SecondAlg : public AlgBase
{
    public:
        SecondAlg(const std::string& name);

        bool initialize();
        bool execute();
        bool finalize();

    private:
        unsigned int  m_count;
        IfMySvc*      m_svc;
};
#endif
```

And the source file is,

```c++
// file SecondAlg.cc
#include "SecondAlg.h"
#include "FirstSvc/IfMySvc.h"
#include "SniperKernel/AlgFactory.h"

DECLARE_ALGORITHM(SecondAlg);

SecondAlg::SecondAlg(const std::string& name)
    : AlgBase(name),
      m_count(0)
{
}

bool SecondAlg::initialize()
{
    LogDebug << "in the SecondAlg::initialize()" << std::endl;

    SniperPtr<IfMySvc> _svc(this->getScope(), "MyService");
    if ( _svc.valid() ) {
        LogInfo << "the IfMySvc instance is retrieved" << std::endl;
    }
    else{
        LogError << "Failed to get the IfMySvc instance!" << std::endl;
        return false;
    }
    m_svc = _svc.data();

    return true;
}

bool SecondAlg::execute()
{
    LogDebug << "in the SecondAlg::execute()" << std::endl;

    ++m_count;
    LogInfo << "loop " << m_count << std::endl;

    if ( m_count%2 == 1 ) {
        m_svc->doSomeThing();
    }

    return true;
}

bool SecondAlg::finalize()
{
    LogDebug << "in the SecondAlg::finalize()" << std::endl;
    return true;
}
```

In order to avoid the retrieval of FirstSvc in each time of the events loop, we retrieve it in `initialize()`, which is executed only once at the beginning of the application. A class template, SniperPtr, is used for the retrival. Then the raw pointer is assigned to the data member `m_svc`.

- the template parameter can be the concrete type of the service (FirstSvc), or the pure virtual base class (IfMySvc). We use the later one for flexibility in this example.
- the 1st constructor parameter indicates where to find the service. In most situations it is `this->getScope()`. It might be different when we have multiple Task instances, which will be described as an advanced topic in later sections.
- the 2nd constructor parameter indicates the name of the service instance;

During the event loop, the service is not always invoked. In this example, it's invoked only when the loop count is odd.


## Python Configuration and Execution

The Python configuration script is,

```python
#!/usr/bin/env python

import Sniper
task = Sniper.Task("task")
task.setEvtMax(3)
task.setLogLevel(3)

import FirstSvc
task.createSvc("FirstSvc/MyService")

import SecondAlg
task.createAlg("SecondAlg")

task.run()
```

And the execution result is,

```
$ python run.py
*****************************************
***     Welcome to SNiPER Python      ***
*****************************************
Running @ lxslc603.ihep.ac.cn on Mon Dec 10 12:58:26 2017

task:SecondAlg.initialize      INFO: the IfMySvc instance is retrieved
task.initialize                INFO: initialized
task:SecondAlg.execute         INFO: loop 1
task:MyService.doSomeThing     INFO: Do some thing in a service
task:SecondAlg.execute         INFO: loop 2
task:SecondAlg.execute         INFO: loop 3
task:MyService.doSomeThing     INFO: Do some thing in a service
task.finalize                  INFO: finalized

***  SNiPER Terminated Successfully!  ***
```

Please notice that we retrieve the service instance via the name "MyService" in C++, which is not a fixed service type. Then in Python, we assigne the name of the FirstSvc instance as "MyService". 

It's simple to implement a new service with the same API, 

```c++
class SecondSvc : public SvcBase, public IfMySvc
{
    // the definations ...
}
```
And it's simple to replace FirstSvc by SecondSvc in the Python configuration,

```python
#import FirstSvc
#task.createSvc("FirstSvc/MyService")
import SecondSvc
task.createSvc("SecondSvc/MyService")
```

In this way, we can flexibly assemble many DLE modules and add/remove/replace each one of them, just by configuration.
