# Gru Workflow Guide v1.0

## What Is Gru

[Gru] is a swiss army ETL and workflow tool to enable a developer to write workflows combining any downstream system into a cohesive set of actions.  

It is customized for the Networked Insights environment and the ETL and workflow systems we have in house.  This way, we can leverage the downstream systems for what they're good at, and shell out to custom processes in a way that is easiest for us (Spring/Java/Mybatis/Postgres/etc).  

This is preferable to relying on one system's plugins to talk to every other system we have in house, or writing brittle java code to piece everything together.  

Finally, having a file-configured workflow allows for easy svn tracking/diffs, and simple deployment, as there is no need to deploy Gru webservice, just your new or updated workflow file.

## How Gru Workflows Work


In Gru, a workflow specifies a number of steps that are to be run in order.  This is useful for grouping together related elements into a cohesive set of procedures with a related purpose - for example, the hourly hive fact load.  

Gru can interact with the following downstream systems:

* PDI
* Jobtrain 
* Greenplum
* Any RESTful webservice
* Borg Oozie
* Gatekeeper 
* Custom spring-aware java/groovy processes inside of Gru


There is a UI to run workflows, and tools-jobs can also be used to cron requests through the REST interface.

For now, all Gru workflows are like singletons in tools-jobs - **only one invocation** of the workflow can run at a time.  If a workflow request comes through while one is currently running, it will halt and not be processed or recorded.

#### Some other features
* branching via if/else statements
* parallel processing via forkSteps
* [gStrings]
* Ability to pass in dynamic parameters per workflow request
* Ability to pass results of steps down to dependent steps
* Custom scriptlets inside of workflow file definition
* Email notification on success and failure, stacktrace of failure included

#### Syntax

Gru uses a DSL that is almost a direct pass through to groovy [object graph builder].  The result is a structured text format that looks similar to a [pretty toString() printout].

## How To Create A Workflow

Workflows live [in svn] next to oozie workflows and get deployed to hdfs via the dcache plugin.  Each file must end with .wf and live in /src/main/resources/apps.  1 file = 1 workflow.

All workflows must provide a workflow block and some meta information.  Gatekeeper locks that will be requested and held the entire workflow may go in the meta block as well.

```
workflow  {
    
    meta (
        name : 'Name of my workflow',
        description : 'Optional description of my workflow',
        email : '<comma delimited list of email addresses>',
        gatekeeperLocks : [
            gatekeeperLock (
                lockType : 'WRITE', 
                lockName : "<lockName goes here>"
            )
            [, gatekeeperLock(...)]
        ]
    )
    
    <Steps go here>
}
```

## Step Types and example invocations

#### PDI

Interacts via the [pentaho kettle client].  Will use the environment base that corresponds to which environment Gru is running.

*Type string:* `pdi`

##### Required: 
* `pdiType`: JOB or TRANSFORMATION
* `name`: name of file on server

##### Example:

```
step (
    name : 'pdi step',
    description : 'pdi step',
    type : 'pdi',
    params : [
        'pdiType' : 'JOB',
        'name' : 'dashboard__brand_profile_transfer.kjb'
    ]
)
```

#### Jobtrain
Interacts via the [jobtrain client].  Will use environment base as well.

*Type string:* `jobtrain`

##### Required:
* `params`: everything in params will be wired in to jobtrain verbatim.
###### Optional:
* `partitionInfo`: partition info to be turned into gatekeeper locks
    
##### Example:
```
step (
    name : 'Jobtrain Post Fact',
    description : 'Load post_fact with newly imported media_recent_v3 data',
    type : 'jobtrain',
    partitionInfo : [
        partitionInfo (
            lockType: 'READ', 
            attribute : "${appProps['oozie.hive.import.table.recent']}.imported_day", 
            format : "yyyy-MM-dd",
            from : 'deferred: { workflow.workflowParams["previousImportedDay"] }',
            to : 'deferred: { workflow.workflowParams["importedDay"] }'
        ),
        partitionInfo (
            lockType: 'WRITE', 
            attribute : "post_fact_${env}.imported_timestamp", 
            format : "",
            from : "1",
            to : "1"
        ),
    ],
    params :  [
        "workflow.project" : "entityloader",
        "workflow.name" : "entityloader-full-post-fact",
        "workflow.version" : "latest",
        "queueName" : "import",
        
        "importedTimestamp" : 'deferred: { workflow.workflowParams["importedTimestamp"] }',
        "lookbackDay" : timeUtils.getLookbackDay(),
        "dedupeLookbackTimestamp" : timeUtils.getDedupeLookbackTimestamp(),
        
        "postFactPointer" : 'deferred: { workflow.workflowParams["postFactPointer"] }',
        "mediaRecentPointer" : 'deferred: { workflow.workflowParams["mediaRecentPointer"] }',
        
        "mediaTable" : appProps['oozie.hive.import.table.recent']
    ]
)
```

#### Greenplum

Call a function or execute arbitrary commands against Greenplum.  Spins up a thin thread and periodically checks until completed.  Calling a function is the default mechanism, and assumes an ordered key/value set of items in params to wire into the procedure.  Calling verbatim commands is turned on via the **runVerbatim** flag in params.

*Type string:* `greenplum`

##### Required

* `procedure`: what to call

##### Optional

* `params`: ordered set of params to be wired into the function if not running verbatim.

##### Example Procedure
```
step (
    name : 'Pull Hive dataview_interest_fact_dashboard into Greenplum',
    description : 'Refresh updated facts from hive',
    type : 'greenplum',
    procedure : 'etl_app.classified_etl_dataview_interest_fact',
    params : [
        '0' : gruProps['days.to.lookback.for.greenplum']
    ],
    gatekeeperLocks : [
        gatekeeperLock (
            lockType : 'READ', 
            lockName : "jobtrain-impl-dataview_interest_fact_dashboard_${env}.authored_date-${timeUtils.getImportedDay()}"
        )
    ]
)
```
##### Example verbatim
```
step (
    name : 'Vacuum Facts',
    description : 'Vacuum facts from hive',
    type : 'greenplum',
    procedure : 'vacuum analyze classification.dataview_interest_fact;',
    params :  [
        "runVerbatim" : "true"
    ]
)
```

#### RESTful Webservice

There are two java step types that can call out to endpoints given a VERB and contentType.  Async fires and forgets, Synchronous will wait until complete, or timeout.

* className: `'com.ni.api.gru.impl.commands.javasteps.reststeps.AsyncRestStep'`
* className: `'com.ni.api.gru.impl.commands.javasteps.reststeps.SynchronousRestStep'`

##### Required params

* `url`: full url to call
* `verb`: GET POST etc
* `contentType`: application/json etc

##### Example
```
step (
    name : 'Generate site comparison report',
    description : 'es-hive comparison report',
    type : 'java',
    className : 'com.ni.api.gru.impl.commands.javasteps.reststeps.AsyncRestStep',
    params : [
        "url" : "http://labs.colo.networkedinsights.com:30099/report?type=Site",
        "verb" : "GET",
        "contentType" : "application/json"
    ]
)
```
#### Borg oozie

Submit a job directly to oozie on borg.

*Type string:* `borg`

##### Required
* `params`: everything that needs to be passed into the oozie job

##### Example
```
step (
    name : 'borg',
    description : 'borg',
    type : 'borg',
    params : [
        'workflow.name' : 'copy-dataview-interest-fact-recent-partitions',
        'workflow.version' : 'latest',
        'workflow.project' : 'analytics-workflows',
        'nameNodeProd' : 'hdfs://nn1prod.colo.networkedinsights.com:8020',
        'queueName' : 'default'
    ]
)
```

#### Custom Java/Groovy

Call a custom step in a spring-aware environment.  In order to create a custom step, make a new class in `com.ni.api.gru.impl.commands.javasteps` and implement `Callable<Map<String,String>> getStepImpl(Map<String,String> workflowParams, Map<String,String> stepParams) `.  The return value is used to populate the workflow params map.  **This is the key for passing parameters to later steps.**

##### Example java
```java
package com.ni.api.gru.impl.commands.javasteps

import java.util.concurrent.Callable

import javax.annotation.Resource

import org.slf4j.Logger
import org.slf4j.LoggerFactory

import com.ni.api.gru.impl.util.Mailer

class EmailingStep extends JavaStepImpl {

	
	static Logger LOG = LoggerFactory.getLogger(EmailingStep.class)

	public String SUBJECT = "subject"
	public String CONTENT = "content"
	public String TO = "to"
	
	@Resource
	Mailer mailer

	@Override
	Callable<Map<String,String>> getStepImpl(Map<String,String> workflowParams, Map<String,String> stepParams) {

		return new Callable<Map<String,String>>() {

			def call() throws Exception {
				mailer.sendMail(stepParams[SUBJECT], stepParams[CONTENT], stepParams[TO])
			}
		}
	}
	
}
```

*Type string:* `java`

##### Required
* `className`: fully package qualified path to your class

##### Optional params
* Anything present in params is available to your custom step
    
##### Example
```
step (
    name : 'Read Pointers',
    description : 'Find pointers to use for the current run from the media_pointers_v3 table',
    type : 'java',
    className : 'com.ni.api.gru.impl.commands.javasteps.ReadPointersAndSetTimeParams',
    params : [
        'pointerTables' : "post_fact_${env},${appProps['oozie.hive.import.table.recent']},dataview_interest_fact_dashboard_${env}"
    ]
)
```

## Make a new step type

In order to make a new step type, one must create a new class with the same characteristics as the others in the `com.ni.api.gru.impl` package, including naming convention.  Simply implement or override `init()`, `fireStep()`, `checkIfStepComplete()`, and `cleanup()`.  

If you have any additional needs for params, you can either use the provided stepParams or add a new param type outside of it, but be mindful that you must wire it up in the database layer as well.  Check out [StepTo] for more information

## Deferred params
Deferred indicates that you'd like Gru to evaluate your snippet at runtime, not during the creation of the workflow request.  This is the key to allowing steps to pass information to later steps.  The same bindings that exist during workflow request creation time exist during evaluation of these params.

#### Example
```
params : [
    'pointerKey' : "dataview_interest_fact_dashboard_${env}", 
    'pointerValue' : 'deferred: { workflow.workflowParams["importedTimestamp"] }'
]
```
This example will write the value of the pointer to be the value of the workflow param named **importedTimestamp**, which was set earlier in the workflow.

## ForkSteps

ForkSteps allows for steps to be processed at the same time.  All steps inside the block will process simultaneously, and the workflow will not continue to the next step or series of steps until all steps in ForkSteps is complete.

#### Example
```
forkSteps {
    step (
        name : 'Write Post Fact Pointer',
        description : 'Write the pointer for post_fact to indicate we\'re caught up',
        type : 'java',
        className : 'com.ni.api.gru.impl.commands.javasteps.WritePointer',
        params : [
            'pointerKey' : "post_fact_${env}", 
            'pointerValue' : 'deferred: { workflow.workflowParams["importedTimestamp"] }'
        ]
    )
    step (
        name : 'Jobtrain Dataview Interest Fact',
        description : 'Load dataview_interest_fact with newly imported post_fact data',
        type : 'jobtrain',
        params : [
            "workflow.project" : "entityloader",
            "workflow.name" : "entityloader-thinslice-dataview-and-interest",
            "workflow.version" : "latest",
            "queueName" : "import",
            
            "suffix" : "_dashboard",
            
            "postFactPointer" : 'deferred: { workflow.workflowParams["importedTimestamp"] }',
            "dataviewInterestFactPointer" : 'deferred: { workflow.workflowParams["dataviewInterestFactPointer"] }'
        ],
        partitionInfo : [
            partitionInfo (
                lockType: 'READ', 
                attribute : "post_fact_${env}.imported_timestamp", 
                format : "",
                from : "1",
                to : "1"
            ),
            partitionInfo (
                lockType: 'WRITE', 
                attribute : "dataview_interest_fact_dashboard_${env}.authored_date", 
                format : "yyyy-MM-dd",
                from : timeUtils.getLookbackDay(),
                to : timeUtils.getImportedDay()
            ),
        ],
    )
}
```

## Using Custom Scripts

A workflow file gets evaluated when parsed, so **custom groovy code is possible**.  While you are stuck with whatever is on the classpath in Gru, anything on the classpath can be imported and used.  Further, some context variables are set in the binding when evaluating, both in a `deferred:` block and when creating the request.

#### Binding variables

* `env`: which environment we're in
* `appProps`: map of application.properties
* `gruProps`: map of gru.properties
* `timeUtils`: handy date/time methods

#### Example: gString for which table to load

```
params : [
    "postFactTable":"post_fact_${env}"
]
```

#### Example: only run a certain step on off-hours

```
import org.joda.time.DateTime

workflow {

    meta (
        name: 'off hour workflow',
        email: 'lucas.gray@networkedinsights.com',   
    )
    
    if (new DateTime().hourOfDay() in [0,1,2,3,4,22,23]) {
        step (
            name : 'off hour step',
            description : 'off hour step',
            type : 'java',
            className : 'com.ni.api.gru.impl.commands.javasteps.EmailingStep',
            params : [
                'subject' : "off hour step", 
                'content' : 'off hour step', 
                'to' : 'lucas.gray@networkedinsights.com'
            ]
        )
    }
}
```

## How To Verify Syntax Of Your Workflow

From the workflows directory,

```
./gradlew parse -Pworkflow=<nameOfWorkflow>
```
Do not use the file extension or the directory, just the name of the workflow.  If the workflow is valid, you will see a message and the object graph will be printed.  If not, you will be shown the descriptive error message.

**Note: this script depends on gru-impl and may need version corrected from time to time!**

## How To Deploy Your Workflow

Use the deploy buttons in jenkins, after merging into the appropriate svn location for that environment.  Gru workflows get deployed separately from gru, and uses the dcache plugin to deploy to hdfs.

#### How To Update This Document

This html is generated from [this markdown file] and uses [this markdown generator] with the mixu-page theme.

[Gru]:http://gru.colo.networkedinsights.com/index.html
[gStrings]:http://groovy.codehaus.org/Strings+and+GString
[in svn]:http://svn.colo.networkedinsights.com/repos/repo/projects/apps/trunk/workflows/gru-workflows/
[pentaho kettle client]:http://svn.colo.networkedinsights.com/repos/repo/projects/warehouse/trunk/services/pentaho-kettle/pentaho-kettle-client/src/main/java/com/ni/api/pentaho/client/KettleClient.java
[jobtrain client]:http://svn.colo.networkedinsights.com/repos/repo/projects/apps/trunk/services/jobtrain/jobtrain-client/src/main/java/com/ni/api/jobtrain/client/JobTrainClient.java
[StepTo]:http://svn.colo.networkedinsights.com/repos/repo/projects/apps/trunk/services/gru/gru-impl/jar/src/main/java/com/ni/api/gru/impl/dao/to/StepTo.groovy
[this markdown file]:http://svn.colo.networkedinsights.com/repos/repo/projects/apps/trunk/workflows/gru-workflows/src/markdown/README.md
[this markdown generator]:https://github.com/mixu/markdown-styles
[object graph builder]:http://mrhaki.blogspot.com/2009/09/groovy-goodness-building-object-graphs.html
[pretty toString() printout]:http://svn.colo.networkedinsights.com/repos/repo/projects/apps/trunk/workflows/gru-workflows/src/main/resources/apps/oneoffHiveDedupe.wf
