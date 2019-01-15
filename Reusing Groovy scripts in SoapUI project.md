# Reusing Groovy scripts in SoapUI project

The task is to avoid duplicating the code by collecting the frequently used Groovy code (scripts, functions) in one place and reuse it in different places of SoupUI project (test steps, assertions etc.).

The proposed solutions is applicable for **free** SoupUI edition.

## Solution 1 – Reuse Groovy script

Solution is convenient for the case when the code is used only once in running test step.

Create the new TestSuite named **ScriptLibrary** under the project, with the structure:

	Project
	|-ScriptLibrary (TestSuite in disabled state)
	  |-Scripts (TestCase in disabled state)
		|-TestSteps
		  |-Script1 ('Groovy Script' TestStep that contains reusable code)
		  |-Script2 
		| ...
		
Suppose, we need to generate the unique [UUID](https://tools.ietf.org/rfc/rfc4122.txt) value for using in different Test Steps of our project (for example, as the _'correlationId'_ parameter of 'SOAP Request' TestSteps).

Add the following code in **Script1** under the **ScriptLibrary\Scripts**:

```java
import java.util.Random

def val = UUID.randomUUID().toString()
context.setProperty('UUID', val)

log.info('UUID=' + val)
```

This script generates the unique value and assigns it to _UUID_ property of project context.

Then we should add 'Groovy Script' TestStep named **InitScript1** under TestCase, as the first TestStep in TestCase:

	|-TestSuite
	| |-TestCase
	|   |-TestSteps
	|     |-InitScript1 ('Groovy Script' TestStep  to initialize library script **Script1**)
	|     |-TestStep (Run TestCase TestStep)

In **InitScript1** we initialize context properties by invoking the appropriate library script **Script1**:

```java
testRunner.testCase.testSuite.project
	.testSuites["ScriptLibrary"]
	.testCases["Scripts"]	
	.testSteps["Script1"]
	.run(testRunner, context)
```

In our case this initializes context property _UUID_ that could be used then in any following TestSteps of the TestCase (for example, as the value of _correlationId_ parameter in 'SOAP Request' TerstStep), in such way:

```xml
<correlationId>${=context.UUID}</correlationId>
```	

This expression will produce, for example, the following output: 

```xml
<correlationId>bfc92a49-0c5e-485c-a819-f8d124524885</correlationId>
```

## Solution 2 – Reuse Groovy class

If some function should be called multiple times within TestSuite, we can include it in new Groovy class, instantiate this class and call function code as a class method.

Suppose, several unique UUIDs with different prefixes should be generated as the values of the _correlationId_, _messageId_ parameters of 'SOAP Request' TestSteps.
Create subdirectory **/ScriptLibrary** in current project directory and add there new file **GroovyScript1.groovy** with such content:

```java
import java.util.Random

class ScriptClass1 {
	// TestSuite environment variables
	def log
	def context
	def runner

	// Class constructor
	ScriptClass1(logIn, contextIn, runnerIn) {
		// store the environment variables (some class methods might need them)
		this.log = logIn
		this.context = contextIn
		this.runner = runnerIn
	}

	// Reusable function
	String UUID(prefix) {
		def val = prefix + UUID.randomUUID().toString()
		this.log.info('UUID=' + val)
		return UUID
	}
}

// instantiate class and assign it to context property
context.setProperty("ScriptClass1", new ScriptClass1(log, context, runner))
```
	
This code defines new Groovy class **ScriptClass1** with class method **UUID** that returns unique UUID value with prefix passed in input parameter of this method. Then the new class instance is created and assigned to _ScriptClass1_ property of context to allow other scripts to reuse the class code. 

To access the class instance elsewhere within SoapUI TestSuite we can add the following code in TestSuite _Setup Script_:

```java
def groovyUtils = new com.eviware.soapui.support.GroovyUtils(context)
def projectDir = groovyUtils.projectPath // Project path 
def scriptPath = projectDir + "\\ScriptLibrary\\GroovyScript1.groovy" // Path to groovy script

evaluate (new File (scriptPath)) //Instantiate class and assign instance to 'ScriptClass1' property of context
```

Then we can call the class method on class instance stored in _ScriptClass1_ property of context, for example, call method **UUID** to generate values of _correlationId_, _messageId_ parameters in 'SOAP Request' TestStep:

```xml
<correlationId>${=context.ScriptClass1.UUID('CorID-')}</correlationId>

<messageId>${=context.ScriptClass1.UUID('MsgID-')}</messageId>
```

These expressions will produce, for example, the following output: 

```xml
<correlationId>CorID-6811b474-e7e7-4979-b169-02fe1893bb83</correlationId>

<messageId>MsgID-d43207a8-dd1f-4c16-9e35-a45c0dd4dbfd</messageId>
```
