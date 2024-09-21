# jenkins-report-jtreg
Jenkins plugin to show unit-test, tesng, jtreg and JCK reports summaries, diffs and details

Plugin is in `report-jtreg` module. `report-jtreg-lib` is code share by plugin and other modules. `report-jtreg-comparator`, `report-jtreg-list`, `report-jtreg-diff` are standalone CLI programs, which can operate overresults. See lower for individual descriptions. See -h/--hel "in themselves".  The `report-jtreg-service` is CGI (yes, cgi) html wrapper around all three cli tools.  Although you can setup `plugin` to link to them (and it is awesome) it is not exactly safe, so we reocmmend to not run those services on public network. IN addition, the `services` are really clumsy to set up, refer to the source code of `io.jenkins.plugins.report.jtreg.main.Service` for details

The plugin reads archived gzipped xml files prdoduced by junit/testng/jtreg/jck suites  ([or anyhow else generated](https://github.com/rh-openjdk/run-folder-as-tests/blob/main/jtreg-shell-xml.sh)) runs.

* [Implementation details](#implementation-details)
* [Job run details](#job-run-details)
    * [run page](#run-page)
    * [details page](#details-page)
* [Project details](#project-details)
* [View](#view-summary)
* [Denylist and Allowlist](#denylist-and-allowlist)
* [Project Settings](#project-settings)
* [View Settings](#view-settings)
* [Limitations](#limitations)
* [Cli tools and services](#cli-tools-and-services)
    * [List](#list)
    * [Comparator](#comparator)
    * [Diff](#diff)
* [Future work](#future-work)
* [For developers](#for-developers)
    * [embedding lib in plugin](#embedding-lib-in-plugin)
    * [releaseing](#releaseing)

## Implementation details
The xml reports you recieve, should be post processed a bit, and compressed. Zip, tar.gz and tar.xz (including xml.gz and xml.xz) are supported. Compressed, as plugin is used with reports hundrets of megabytes large. And postprocessed as various engines generates various reports. Thus the xml files should be gathered into archives, which are later considered as suites:
![suites](https://user-images.githubusercontent.com/2904395/43016538-6c5141aa-8c53-11e8-8b6f-2eb45ebcaf01.png)
The level of granularity is up to you. The tar.gz archvies are later cached as two relatively small json files - one with listing for diff, one with stack traces of failures.  Latest cache is the properties file with total/run/passed/failed/error/skipp keys to be reused via https://github.com/judovana/jenkins-report-generic-chart-column (and used for quicker renderig of graphs of this plugin itself)

**Note, that jtreg parser is reading for jtr.xml files, not just .xml, because we hit issue that ntot all xml files in results archives are desired. Thus, if you are using this to parse junit.xml compatible files, renam,e them to jtr.xml. TODO, we really should make this configurable.
**
## Job run details

### run page
Quick overview in build page:
![koji-jtreg-rpms](https://user-images.githubusercontent.com/2904395/43510354-a8f3e86c-9575-11e8-9318-c3516d65a876.png)
simple, but a lot of saying ```JTREG Reports Total: X, Error: Y, Failed: Z``` message is printed, where **JTREG Reports** is link to  dretails page:

### details page
![details page](https://user-images.githubusercontent.com/2904395/43016541-6cb4cc5c-8c53-11e8-944b-cf1d274c492e.png)
Here yo can see several items:
* dummy **navigation** which allwos you to skip back and fwd. It do not check status of run, so it can return 404 on failed run
* **table**, with suites. Each suite have detailed  total/passed/failed/error skip details.  Table is followed by
* **listing of faiures**. The listing is clear, and contains failed tests only. Each failure contains expandable
    * **trace**. The listing is followd by
* **diff**. Diff is done first for addereport-jtreg-service/target/report-jtreg-service.jard/removed suites. For unchanged suites, the diff of suite itself is done. You can immediately see exact **fixed** and **freshly failing** tests. After the diff, you can see 
* **full listing** of testsuite. The ful listing is stripped to 1000 lines due to performance. The output is generated by the rutines of [Diff cli tool](#diff-cli-tool), so please apologise a bit of cryptic "passed or missing" and "faile dor error" output. To see the full listing, you must use the [cli](#diff-cli-tool).
* on start of each section is cross roada allowing direct jumps to table, list of issues, diff and full listing
## Project details
![project details page](https://user-images.githubusercontent.com/2904395/43016540-6c95e4a4-8c53-11e8-8e59-db5b6b729b6b.png)
On the project page, you can see several graphs: 
* **number of failures**  shows how much test failed and how much had error. The graph is scalled, but in usual world, there is very small number of failures, and similarly small number of errors, so scale is not affected.
* **number of total tests**  shows how much test exists in suite and how much wasrun. The graph is scalled, but in usual world, there is very small number of skipped tests so both lines are of same value and so scale again should not be affected.
* **regression chart**  shows how much test had changed status. Green bar - number of tests which get fixed. Red bar, number of tests which started to fail. This graph is here to prevent overlook of X fixed tests and (same) X of freshly failing tests. IN such case, your total number of passes/failures is constant, and thus invisible on normla charts.

Each chart have detailed tooltip  and is capable of click which takes you directly to [details page](#details-page):
![tooltip](https://user-images.githubusercontent.com/2904395/43016542-6cd42700-8c53-11e8-9406-e0b3a908c60a.png)

## View summary
You can place jtreg charts also to jenkins view, so you can eyball all your testruns in one gaze.
![view summ](https://user-images.githubusercontent.com/2904395/43015875-21c739fc-8c51-11e8-9026-c84127628634.png)
Here the jtreg plugin is the left most graph. The two right most graphs are [charts from properties](https://github.com/judovana/jenkins-report-generic-chart-column)

## Denylist and Allowlist
You could have noted, that the graphs are scaled. Sometimes it happens that fails 100x more tests then usually. This is killing the scale, and you can miss the regression. Such a build deserves  to be denylisted once the issue is solved. On contrary, allowlist is here to allow you to comapre just some selected runs. Both lists are space separated list of regexes against job name (usually #NUMBER or some_custom_name). The lists are shared betwen project and view.

## Project Settings
![jtreg-project-settings](https://user-images.githubusercontent.com/2904395/43445509-dadbf3da-94a6-11e8-869b-44242a5a20fb.png)
Project settings are simple - you set glob expression for files to be considered as archives with results, and select how many tests you wish to show

## View Settings
![jtreg-view-settings](https://user-images.githubusercontent.com/2904395/43445508-dabd0682-94a6-11e8-824b-2609128dd016.png)
View settigns are fully inherited from project settings. So the only thing you do in view is to set the order - column for a chart

## Limitations
The imitations are clear from shared settings with all pros and cons and quite clumsy comparsion of exact jobs (w/b lists) and impossible comparsion between jobs.

## Cli tools and services
FIXME
To workaround limitations, and add possibility to post-process results, the hpi and jars contains main class of ``` io.jenkins.plugins.report.jtreg.main.list.CompareBuilds``` whcih allows (based on the director with yor jobs) to comapre or list practically anything.  The launcher can look like:
```
set -xeuo pipefial
CP=report-jtreg-lib/target/report-jtreg-lib.jar:report-jtreg-$MODULE/target/report-jtreg-$MODULE.jar # for module, se below
#CP=$CP:report-jtreg-service/target/report-jtreg-service.jar #if you want to run it as service FIXME
/opt/jdk/bin/java -cp $CP -Djenkins_home=$jenkins_main_home  $MAIN  $@ # for individual CLI main methods see lower. The service is io.jenkins.plugins.report.jtreg.main.Service
```
It can spwn plain tex, colored tex, or even html, so it i easy to be deployed as service (there is a wrapper for this too - ```Service``` ) together with  jenkins.
```
All cli/services return their switches if no parameter or -h/--help is present
```
### List
module `report-jtreg-list`, main class: `io.jenkins.plugins.report.jtreg.main.list.CompareBuilds`

The cli works with absolute and relative job IDs, and strictly cooperates with stdout/err (so consider  2>/dev/null sometimes)
* with no param it prints name
* with no job it prints jobs
* with no build it lists builds (of given job)
On everything else it do the listings and/or comparsion (or fails). Eg summary:
![cli1](https://user-images.githubusercontent.com/2904395/43448360-0b965e6e-94ae-11e8-986e-24faba2a8cce.png)

List is capable also of basic diff:
![cli-dif1-cli-dif3](https://user-images.githubusercontent.com/2904395/43448359-0b793672-94ae-11e8-9285-a837f4a4b8b9.png)

html view:
web-cli - ```Service``` - is nothing more then wrapper around ``` io.jenkins.plugins.report.jtreg.main.list.CompareBuilds``` and is doing nothing more then resending stdout/err to browser request!! There is hardcoded port of 9090 in Sevice class.
![wb1-web3](https://user-images.githubusercontent.com/2904395/43450562-4346fab2-94b3-11e8-931a-bb26456d8aac.png)
Html output is much more clumsy, but the listing of switches and jobs is live, and also ajax is helping here a bit. Also yu can send results as URL, so it have its cases

### Comparator
module `report-jtreg-comparator`, main class:  `io.jenkins.plugins.report.jtreg.main.comparator.VariantComparator`

FIXME

copares individual variants of test runs

It compares by both status, and stack trace. 

### Diff
module `report-jtreg-diff`, main class: `io.jenkins.plugins.report.jtreg.main.diff.StackTraceDiff`

FIXME

shows diffs in stack traces. Mostly used by [Comparator](#comparator)

## Future work
* To figure out how to make job-name based view on top of the shared settings.
* make cli more user friendly

This plugin depends on https://github.com/judovana/jenkins-chartjs-plugin

## For developers
### embedding lib in plugin
In the compile phase of the `report-jtreg-plugin` module a `script.sh/.bat` is run. The script copies the `target` directory of the already compiled `jtreg-report-lib` module into the `target` directory of the plugin module.
This is done because of the Jenkins security hardening (https://www.jenkins.io/blog/2018/03/15/jep-200-lts/). The plugin can't load classes from external modules, and it throws a class filter exception.

The plugin should work correctly with this "workaround", however, if it doesn't, run Jenkins with this switch, it should solve the problem: `-Dhudson.remoting.ClassFilter=io.jenkins.plugins.report.jtreg.model.Report,io.jenkins.plugins.report.jtreg.model.ReportFull,io.jenkins.plugins.report.jtreg.model.Suite,io.jenkins.plugins.report.jtreg.model.Test,io.jenkins.plugins.report.jtreg.model.TestOutput`

### releaseing
This plugin is **multimodule** lib, plugin and two external services. Thus the autorelease do not work as expected.
To release, you have to `cd report-jtreg` submodule which contans `report-jtreg-plugin artifactId` and here run manually `mvn release:prepare` and `mvn release:perform`. During preapare, always adjsut all modules, not jsut dependent ones (choice of `0`). 

Sometimes, you may need to fake the build bit, and to change project version to not-snapshot one (the upcoming release), and  `mvn clean install` lib, so it is in local m2 repos. then rewert the change, and proceed as described above.

After release, you may need to fix your poms, eg:
```
diff --git a/pom.xml b/pom.xml
index ccca4f6..3cb2495 100644
--- a/pom.xml
+++ b/pom.xml
@@ -25,7 +25,7 @@
     </modules>
 
     <properties>
-        <revision>2.6</revision>
+        <revision>2.7</revision>
         <changelist>-SNAPSHOT</changelist>
         <gitHubRepo>jenkinsci/report-jtreg</gitHubRepo>
         <chartjs.version>1.0.2.6</chartjs.version>
diff --git a/report-jtreg/pom.xml b/report-jtreg/pom.xml
index f357a9a..7eea371 100644
--- a/report-jtreg/pom.xml
+++ b/report-jtreg/pom.xml
@@ -37,7 +37,7 @@
         <dependency>
             <groupId>io.jenkins.plugins</groupId>
             <artifactId>report-jtreg-lib</artifactId>
-            <version>2.6</version>
+            <version>${revision}${changelist}</version>
             <scope>provided</scope>
         </dependency>
     </dependencies>
```
Because vhere mvn release:prepare and release:perform can remove the variables properly, it can not restore them as expected:
https://github.com/jenkinsci/report-jtreg-plugin/commit/fe6d56b43304a41c85ec1d4eea965d257891e8cf
-> https://github.com/jenkinsci/report-jtreg-plugin/commit/5bebd159ef746604dcddfbb250517cddbf6723f5 (amended)

