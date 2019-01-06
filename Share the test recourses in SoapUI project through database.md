# Share the test recourses in SoapUI project through database

The task is to share the test resources (аor example: share IP address, phone numbers, card codes in allocated ranges, etc.) across test team members to assure that each resource will be used only once.

One of the possible solutions is to use the database for resource sharing.
Let's suppose that several testers need to generate the IP address from allocated range to use in SoupUI 'SOAP Request' TestStep. For simplicity, we assume that the "range" of IP addresses is configured by an initial value, which is incremented on each request.

The value of next available IP address is stored in Oracle database: table PROJECT contains root record for all project parameters, which are stored in table PARAMS, as shown on the diagram:

	|-------------------|                   |-------------------|
	| PROJECT           |                   | PARAM             |
	|-------------------|                   |-------------------|
	| ID NUMBER(20) PK  |<------------------| ID NUMBER(20) PK  |
	| NAME CHAR(50)     |                   | NAME CHAR(50)     |
	|-------------------|                   | VAL CHAR(200)     |
                                            | PREV_VAL CHAR(200)|
                                            |-------------------|

Inside SoapUI project we create the new TestSuite named **ScriptLibrary** under the project, with the structure:

	Project
	|-ScriptLibrary (TestSuite in disabled state)
	  |-Scripts (TestCase in disabled state)
		|-TestSteps
		  |-GetIPAddress ('Groovy Script' TestStep for requesting the IP Address from database)
		  | ...


**GetIPAddress** script under the **ScriptLibrary\Scripts** contains a such Groovy code:

```java
import groovy.sql.Sql

def conn_str = "jdbc:oracle:thin:user_name/password@hostname:port/service_name"

def query_select = "SELECT val FROM param pm\
 LEFT JOIN project pj ON pm.ID = pj.ID \
 WHERE pm.name = 'SOAP_REQUEST_NAME.IP-ADDRESS' AND pj.name = 'PROJECT-NAME';"
 
def query_update = "UPDATE param pm SET val = ?, prev_val=?\
 LEFT JOIN project pj ON pm.ID = pj.ID\
 WHERE pm.name = 'SOAP_REQUEST_NAME.IP-ADDRESS' AND pj.name = 'PROJECT-NAME';"

def sql = Sql.newInstance(conn_str)

def  res = sql.firstRow(query_select )  /* actual IP address value */

def ip = res[0]
log.info "ip_old= " + ip

(s1, s2, s3, s4) = ip.tokenize( '.' )

def v1 = s1.toInteger()
def v2 = s2.toInteger()
def v3 = s3.toInteger()
def v4 = s4.toInteger()

v4 += 1
if (v4 == 256) {
	v4 = 0
	v3 +=1
	if (v3 == 256) {
		v3 = 0
		v2 +=1
		if (v2 == 256) {
			v2 = 0
			v1 +=1
		}
	}
}

def ip_new = v1.toString() + '.' + v2.toString() + '.' + v3.toString() + '.' + v4.toString()
log.info "ip_new= " + ip_new

try {	
	sql.execute(query_update, [ip_new, ip])

	//sql.commit()
	sql.close()
}  
catch (Exception) {
	ip_new = ip
}

context.setProperty('ipAddress', ip_new)
```

This script interrogates database for the first unused value of IP address, assigns it to _ipAddress_ property of project context and increments the value of IP address in database.

Connection to Oracle database requires installation of JDBC driver in SoapUI:

* Download JDBC driver for your Oracle database version from [Oracle web](https://www.oracle.com/technetwork/database/application-development/jdbc/downloads/index.html). For example, with Oracle Database 11.2.0.4 you can use the [ojdbc6.jar](https://www.oracle.com/technetwork/database/enterprise-edition/jdbc-112010-090769.html).
* Store the driver in \lib and bin\ext\ subdirectory of SoapUI installation directory, for example, on Windows with default installation copy driver to:
C:\Program Files (x86)\SmartBear\SoapUI-5.4.0\lib\
C:\Program Files (x86)\SmartBear\SoapUI-5.4.0\bin\ext\

Then we should add 'Groovy Script' TestStep named **InitResources** under TestCase, as the first TestStep in TestCase:

	|-TestSuite
	| |-TestCase
	|   |-TestSteps
	|     |-InitResources ('Groovy Script' TestStep to initialize library script **Script1**)
	|     |-TestStep (Run TestCase TestStep)

In **InitResources** we initialize context properties by invoking the script **GetIPAddress**:

```java
testRunner.testCase.testSuite.project
	.testSuites["ScriptLibrary"]
	.testCases["Scripts"].testSteps["GetIPAddress"]
	.run(testRunner, context)
```

This initializes context property _ipAddress_ that could be used then in any following TestSteps of the TestCase (for example, as the value of _ip-address_ parameter in 'SOAP Request' TerstStep), in such way:

```xml
<ip-address>${=context.ipAddress}</ip-address>
```	

