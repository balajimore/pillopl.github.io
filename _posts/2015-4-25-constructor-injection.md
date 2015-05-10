---
layout: post
title: Why injecting by constructor should be preffered?
---

Seeing the famous <a href="http://google-collections.googlecode.com/svn/trunk/javadoc/com/google/common/annotations/VisibleForTesting.html">@VisibleForTesting</a> guava's annotation for the first time made me thinking. Without looking at the implementation I was pretty sure it is magical piece of code which silently makes our properties visibile only in tests without breaking carefully (?) created encapsulation. Later on, I decided to look at the implementation and javadocs. Here goes the latter one:

<blockquote class="cite">
      <p>An annotation that indicates that the visibility of a type or member has been relaxed to make the code testable.</p>
</blockquote>

#### Problem

**requirement:** supply jar archive with version.info file containing current build version  
**solution:**  

```groovy
/* build.gradle */

apply plugin: 'java'

ext.buildVersion = "1.01-${new Date().format('yyMMddHHmmss')}"

task createVersionFile {
    ant.echo(message: "${buildVersion}", file: "${buildDir}/version.info")
}

jar {
    dependsOn createVersionFile
    from fileTree(dir: "${buildDir}", include: 'version.info')
}
```  
  
Running **gradle build** shows that everything works as expected. Work is done and developer victorious,
but only until another task is run: **gradle test**
with unexpected outcome: new **version.info** created though no jar file was built.

&nbsp;

With **gradle build** situation is trivial cause **build** task indirectly depends on **createVersionFile** task

![gradle build dependencies]({{ site.baseurl }}/images//2015-4-27-gradle-build-phases/gradle-build-tr.png "gradle build dependencies")

&nbsp;

But suprisingly with **gradle test** no such dependency exists.

![gradle test dependencies]({{ site.baseurl }}/images//2015-4-27-gradle-build-phases/gradle-test-tr.png "gradle test dependencies")  

&nbsp;

For experienced gradle user mistake in the script is easy to spot (will be shown at the end),
but still it's not that obvious what has happened.  

#### Build Phases

Gradle has two [build phases](http://gradle.org/docs/current/userguide/build_lifecycle.html) (actually three, but we'll skip initialization)  
- **Configuration** - where all the project objects are configured **=** where the given gradle groovy script is read and executed to build the model  
- **Execution** - where from the model (built in the previous phase) only subset of tasks is executed 

In our **gradle test** case - version file wasn't created on execution phase - but beacause of the simple mistake - already on configuration.

Configuration phase is about executing groovy script found in **build.gradle** - let's see what was done here.  
After applying **java plugin** and setting **buildVersion**, **createVersionFile** task is created.  
What for people not familiar with groovy syntax may look like java method definition, is actually execution of **task()** method with two arguments: **taskName** and **closure**.  
This part - beacuse of groovy flexibility - may be rewritten as:  

```groovy
task (
	createVersionFile, // first argument - taskName
	{ant.echo(message: "${buildVersion}", file: "${buildDir}/version.info")} // second argument - closure
)
```  
  
  
Groovy script is executed on **Project** delegate, 
whose most logic is handled by **org.gradle.api.internal.project.AbstractProject** class, where we can found method

```java
public Task task(String task, Closure configureClosure) {
	return taskContainer
			.create(task)	// new task created
			.configure(configureClosure);	// closure executed on Task delegate
}
```

As we can see closure is immediately executed on new task to configure it - that's why the version file was executed as soon,
as the task was created and not later on it's execution.

#### Actions

Gradle tasks ared designed as collections of Actions (executed on Execution phase), that can be added with **doLast()** method:,
so in our case proper configuration will be:

```groovy
task createVersionFile {
    doLast {
        ant.echo(message: "${buildVersion}", file: "${buildDir}/version.info")
    }
}
```

Here  
- task method was executed again on **Project** delegate,  
- closure was passed to new task's configuration, but this time it was the *doLast()* method, executed with one argument: closure, which was stored as the new action,
all of which can be seen in  

```java
/* org.gradle.api.internal.AbstractTask */

public Task doLast(final Closure action) {
	(...)
	actions.add(convertClosureToAction(action));
	(...)
}
```  

#### Final Solution

It must also be mentioned that **doLast()** method is so popular that there is a shortcut to it - **leftShift(final Closure action)** - also to be found in **AbstractTask** class,
so **createVersionFile** task creation can be rewritten simply as:  

```java
task createVersionFile << {
    ant.echo(message: "${buildVersion}", file: "${buildDir}/version.info")
}
```

&nbsp;

And now everything works as suppose to. 

&nbsp;
****



