### Python XML analysis

In this lab, you will learn to trace a python application using lttng library `lttngust`, write an xml analysis file to create your own view, and visualize your application with your own tracepoints.

*Pre-requisites*: Have Trace Compass installed and opened. Have git, lttng, Python 3.X, pip and the callstack add-on on Trace Compass installed. You can follow the [Installing TraceCompass](../00-installing-tracecompass.md) lab or read the [TraceCompass website](https://tracecompass.org) for more information.

- - -

#### Sub-task 1: Writing the code

For this lab, you will be using a web server example using Flask. This server makes use of the `lttngust` library. To use the lttng ust agent you have to import the LTTng-UST Python agent:

```python
import lttngust
```

The agent automatically adds its logging handler to the root logger at import time. By adding this handler, every log statement creates an LTTng event (more details available in the [LTTng documentation](https://lttng.org/docs/v2.10/#doc-python-application)). The goal in this lab is to track the behaviour of the server, therefore we will create an event before and after each time the server receive a request. In order to do that we will create two functions `tracing_entry` and `tracing_exit` and add them to the flask built-in handlers:

```python
def tracing_entry():
    app.logger.info("route_entry")

def tracing_exit(response):
    app.logger.info("route_exit")
    return response

[...]

if __name__ == "__main__":
    app.before_request(tracing_entry)
    app.after_request(tracing_exit)
    app.run(port=5000)
```

- - -

#### Sub-task 2: Writing the XML analysis

A small explanation on how to write a stack trace xml analysis will be provided here however the documentation on how to write an XML analysis is available in the [Trace Compass user documentation](http://archive.eclipse.org/tracecompass/doc/stable/org.eclipse.tracecompass.doc.user/Data-driven-analysis.html#Data_driven_analysis). 

The XML analysis is used to generate a state system which can track the states of different elements over the duration of a trace using the different events and their properties.
An empty file, with no content yet would look like this:
```XML
<?xml version="1.0" encoding="UTF-8"?>
<tmfxml xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="xmlDefinition.xsd">
    [...]
</tmfxml>
```
In our case we want a callstack which has two elements:
* The `callstackGroup` have metadata telling where in the state system, the callstack state will be stored.
* The `pattern` is the xml element telling Trace Compass how to parse a pattern of trace events in order to build one or multiple state machines. These state machines are used to gain state information of a process, thread or program, from the events.

```XML
<callstack id="lttng.ust.pythonRequest.analysis">
    <callstackGroup name="Python threads">
        <level path="Threads/*"/>
        <thread cpu="cpu"/>
    </callstackGroup>
    <pattern version="1" id="ca.polymtl.pythonrequest">
        [...]
    </pattern>
</callstack>
```

In the pattern element we have 3 elements:
* The `head` which contains the traceType that tells Trace Compass which trace type this analysis apply to, in this case this analysis applies to ust traces. The label indicates the name under which the views will be displayed in the project explorer.
* The `location` indicates how each thread state is displayed in the views.
* The `patternHandler` is where `test`, `action` and `fsm` elements will be defined.
```XML
<head>
    <traceType id="org.eclipse.linuxtools.lttng2.ust.tracetype"/>
    <label value="Python Requests View"/>
</head>

<location id="CurrentThread">
    <stateAttribute type="constant" value="Threads"/>
    <stateAttribute type="query">
        <stateAttribute type="constant" value="#CurrentScenario"/>
        <stateAttribute type="constant" value="threadName"/>
    </stateAttribute>
</location>

<patternHandler>
    [...]
</patternHandler>
```
The `patternHandler` element is where the core of the logic is defined, details about each element are available in [Writing the XML pattern provider](http://archive.eclipse.org/tracecompass/doc/stable/org.eclipse.tracecompass.doc.user/Data-driven-analysis.html#Writing_the_XML_pattern_provider).

* The `test` element defines a condition used in the state machine to execute a transition.
* The `action` element defines a change in the state or a segment generation to apply when there is a transition.
* The `fsm` element defines the states and their transitions.

```XML
<test id="same_thread"> [...] </test>
<test id="new_request"> [...] </test>
<test id="end_request"> [...] </test>

<action id="entering_thread"> [...] </action>
<action id="push_event_type"> [...] </action>
<action id="pop_event_type"> [...] </action>

<fsm id="pythonRequest" initial="Wait_start"> [...] </fsm>
```

Here is the XML that defines the finite state machine (FSM). In this case a request has three states for a server processing it: not started, being processed, finished. These states are respectively `Wait_start`, `in_thread` and `end_thread`. Each state can have transitions to other states and these transitions have conditions in order to parse the trace adequately. They also have an action to execute when the transition happen.
```XML
<fsm id="pythonRequest" initial="Wait_start">
    <state id="Wait_start">
        <transition event="*" cond="new_request" target="in_thread" action="entering_thread:push_event_type"/>
    </state>
    <state id="in_thread" >
        <transition event="*" cond="same_thread:end_request" target="end_thread" action="pop_event_type"/>
    </state>
    <final id="end_thread"/>
</fsm>
```

- - -

#### Sub-task 3: Tracing the python application

To run the server script given in this lab you need to install the lttngust and flask libraries.
```bash
$ pip install lttngust flask
# If you have ubuntu, it may be easier this way:
$ python3 -m pip install lttngust flask
```

In order to record a trace with python events you need to create an lttng session and enable python events:

```bash
# In console 1:
$ lttng create python
$ lttng enable-event --python -a
$ lttng start
$ python3 server.py

# In console 2:
$ curl localhost:5000 && curl localhost:5000/resource1 && curl localhost:5000/resource2 && curl localhost:5000/resource3 && curl localhost:5000/resource4

# In console 1:
# ctrl-c to stop the server
$ lttng stop
$ lttng view # Optional, displays the events recorded
$ lttng destroy
```
If the server takes too long (5 seconds or more), it is possible that the LTTng agent could not register with the session daemon and in that case, the trace will not contain any event. If you run python in a virtual environment, it could prevent the agent from connecting to the session daemon.

- - -

#### Sub-task 4: Running the XML analysis

After importing the trace into Trace Compass, you need to apply an XML analysis on a ust trace to enable certain views to display the state you want to analyze. To enable the XML analysis in your tracing project, you need to right click on the folder above your trace in the project explorer. Then select `Manage XML analyses...`, import your xml file, apply and close the window.

![ManageXML](screenshots/manageXML.png "Trace Compass Manage XML")

![ApplyAndClose](screenshots/applyClose.png "Trace Compass Apply and Close XML")

Once these actions are done, you can open the *Flame Graph View*, under the `Python Views`, to visualize the different queries that have been executed on the python server.

- - -

#### Sub-task 5 (optional): Debugging the state system

Sometimes, when working with XML analysis, things don't work out the way you want them to. One very useful tool to use in this case is the *State System Explorer View*. This view displays the state internal values with respect to time for each opened trace in Trace Compass. It can be opened via the *Window* > *Show View* menu and searching for *State System Explorer*.

![StateSystemExplorer](screenshots/stateSystemExplorer.png "Trace Compass State System Explorer")

- - -

#### Conclusion

In this tutorial, you wrote some python code to create LTTng events, then you traced that application to analyze its behaviour. You also wrote an XML analysis file to make Trace Compass able to parse correctly your python events. Finally you used Trace Compass in order to display the state of your applicaton with respect to time.
