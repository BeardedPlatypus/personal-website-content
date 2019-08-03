title: TeamCity: Adding a custom report to your build configuration
date: 2019-08-03

Recently, I took some time to add a bit of custom reporting to the teamcity 
project we use at work. I did not have much experience with this before, and
honestly, I was quite surprised with the ease to do so. As a reference for 
myself, and hopefully for you as well, in this short article I will detail how I
achieved this.

# Motivation

To give a little bit of background, at the time of writing, I work at Deltares,
a knowledge institute on Water. I work on one of the GUIs for our water 
modelling software. We use TeamCity on our build server. The TeamCity project 
contains a bunch of configurations doing different tests and builds our 
installers.

One of these configurations is our Acceptance Tests configuration. This
configuration runs a bunch of models, made within deltares, that our considered
representative of models that can be made. As such, if the software runs 
correctly for these models, it should run correctly for all models.
The test configuration is set up to run a set of NUnit tests that load these
models, does a few checks and runs the models. Arguably this is not an ideal
set up, but it is the process we have at the moment. 

Each model run produces a log file, that describes what has happened, as well as
any warnings and errors. This file is of interest when these acceptance models
start failing. However, the current set up removes all of these files. We want
to be able to inspect these files, to check them for any warnings or errors that
might point to mistakes in our code (or the kernel code).

A first step to achieve that, would be to save these files in a separate 
artifact. Which allows us to download them, and inspect them. But why stop 
there? We could easily extend this, to allow us to inspect the files from within
TeamCity, which lowers the barrier to do so, quite a bit. We can do so with 
custom reports, hence this article.

# Bring out the custom reports!


## What do we need?

A custom report is basically an additional tab, in which we can display 
additional information, which is produced as part of our build procedure. This
can be anything, really. It can be used to track performance times, it can be 
used to build a custom coverage tool. All we need to add this custom report to
our TeamCity configuration, is an html file. This html file will be hosted in
the TeamCity report tab, [see][https://confluence.jetbrains.com/display/TCD10/Including+Third-Party+Reports+in+the+Build+Results].
You can even put your html file in an archive like zip, and just refer to it.
Something we will do. 

In order to make this report available to our tab, we do need to add it to our
artifacts. As such, the only thing you need is an html file that is published 
as an artifact.

## Optionally adding an additional build step

*Feel free to skip this part if you do not need to add a new build step that executes a python tool*
As mentioned before, the html file should be produced as part of the build
process. If none is produced at the moment, it might be necessary to add another
build step to your configuration, which is responsible for building the actual 
report.

In my case, the tests produce a bunch of log files, one per model run. These
are saved within the check-out folder, such that they will not be immediately
deleted. However, no report html is generated automatically. To do this, I have
written a small python script, that collects the content of the log files and 
stuffs them inside a simple html file. This html file is the one we want to 
publish. Lastly, the script zips the whole set of log files and the index
file into a single zip, for easy download.

In order to execute this script, a build step is added to the configuration: 

1. First navigate to the build configuration, in which we want to produce the
   report. Press the *Edit Configuration Settings* link in the top right corner
   next to the *run* and *actions* button. From here, select the *Build Steps*
   link on the left hand side of the screen.
   
2. We are presented with several buttons that allow us to modify our build
   configuration. We want to press *Add build step* to create our new build
   step at the end of our configuration. 
   
3. You are presented with a wizard to set up your build step. In our case, we
   want to execute a python script, which we will do from the Command Line, 
   thus we select *Command Line* as our runner type, did not see that coming
   did you? 
   
4. Choose an appropriate *Step name*, this will show up in the build steps
   page we were on earlier. 
   
5. In our case we want to execute the python script regardless of whether the
   test runs were successfull, so instead of *If all previous steps finished successfully*
   we can select, *Even if some of the previous steps failed*.

6. We can leave the *Working Directory* empty if you are running from the
   check-out folder, otherwise you want to set the folder in which you want
   to execute your command line script. 
   
7. We can also leave the *Custom script* bit, and move on to filling in the
   build script content.  
   On our build server we have installed `conda`, which we will use to quickly
   generate a python3 environemnt, which we will use to execute the script.
   Then we will run the script, and finally we will remove the environment 
   again, because it is always good to clean up after yourself.  
   This leads to the following script:  
   ```
   CALL conda create -y -n <someEnvName> python=3.5
   CALL activate <someEnvName>
   CALL python "path/to/your/script.py" [withOptionalArguments]
   CALL deactivate
   CALL conda remove -y -n <someEnvName> --all
   ```  
   Of course, you want to replace `<someEnvName>` with your actual environment
   name, and the python call with the one that actually executes your script.
   
We now got our reporting tool set up, which will produce our output somewhere
in our check-out folder (hopefully).

## Adding the report to the artifacts

Assuming we have our reports being generated somewhere within the check-out 
folder. We will have to add them to our artifacts. 

1. First navigate to the build configuration to which we want to add our report
   tab. Then press *Edit Configuration Settings* next to the *Run* and *Actions*
   buttons.
2. On the page that opens there should be an *Artifact paths* section. We can 
   add the path to our report here, such that it gets released as an artifact.
   The given path should be relative to the check-out folder.  
   In our case, the report is made in an *Artifacts* folder, and is called
   *dia_report.zip* (naming has never been one of my strong suits). Thus our
   artifact becomes.  
   ```
   Artifacts\dia_report.zip
   ```  
   If you already have a run that produced the report, you can also press the
   little tree button next to the text field, to select it, producing the 
   correct path.  
   With that done, you should see your report in the next run as an artifact.

## Adding a custom build report

The last step is adding the actual custom report tab to the build configuration.
This is not done within the build configuration, something I found a bit counter
intuitive, but instead should be done at the project root. 

1. Navigate to the `<root project>` and select *Edit Project Settings* in the 
   top right corner.
2. Navigate to the *Report Tabs* page through the menu on your left hand side.
3. There are two options, you can create a new project report tab, or a build
   report tab. Project report tabs will show on the project, and use the 
   artifacts produced by one of the build configurations within the project.
   The build tab is what we want. It allows us to set a path to a report html.
   Any build configuration that contains an html file within its artifacts at 
   this specific path, will have this build tab.
   
   As such, press the *Create new build report tab*
4. Now set the *Tab Title* and the *Start page* of your report html. 
   Press save, and voila! We should have ourselves a custom report tab.

# Conclusion

Within this article we added a simple custom report tab to one of our TeamCity
build configurations. We can use this to provide easily provide additional 
test information directly within our build server. The process itself is 
rather painless, and for the most part intuitive. All things considered, 
I am really happy how this turned out, and I hope it helps you in your 
process as well.

Hopefully see you next time!
