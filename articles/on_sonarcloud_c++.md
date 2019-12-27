title: SonarCloud: Static Code Analysis in a C++ project.
date: 2019-12-27

As part of my pacman project's Continuous Integration (CI), I have set up 
SonarCloud as a static code analysis tool. This was done to get a bit better
insight in the state of my code base, as well as a way to get feedback and
improve my C++ knowledge. Because this ended up being slightly more work than 
I initially had hoped, I will use this article to explain how I set up 
SonarCloud in conjunction with my pacman project. Hopefully this will both 
serve as a reminder to myself and a simple tutorial for you.

For a tl;dr, the DevOps pipeline that is set up can be found [here](https://github.com/BeardedPlatypus/PacMan/blob/7127d4b26988f3442b811a2225583e775bc7b0d9/sonarcloud-pipeline.yml). 
The full pacman project can be found [here](https://github.com/BeardedPlatypus/PacMan).

# Introduction

## Motivation

At my current place of work, we use SonarQube to get insight in the quality of
our code. It provides a simple, insightful dashboard that highlights
bugs, code smells, code coverage and duplication. This allows developers to 
gain insight in the quality of the code they have committed. To me personally,
I love the way certain bad habits get highlighted, and I get forced to fix 
them. Often, there exists some exotic intricacy within the language I am not 
aware of, or I am using in a wrong way, and SonarQube will highlight it 
mercilessly. This allows me to learn and become a better coder in my opinion, 
in much the same way as I learn from ReSharper suggestions.

At work, I was not involved in setting this tool up, so I did not have any 
experience, before I set it up for this project. It turned out to be a bit more
of a struggle than I would like to admit, so a write up was in order.

## Context

It might be useful to give a bit more context in how my project is currently 
set up. This could help evaluate whether the approach I took could be useful 
for your project.

My [pacman clone](https://github.com/BeardedPlatypus/PacMan) is developed in (modern) C++, with the help of the [SDL2 library](https://www.libsdl.org/download-2.0.php)
for rendering the sprites. It is build with visual studio 2019 / MSBuild. 
Testing is done with VSTest and the google test adapter. gtest and gmock are 
used to write the unit tests. All of these libraries are installed through 
vcpkg. The whole thing is wrapped into an installer with the help of WiX.

On the CI side of things, I use Azure DevOps to automate my build and test 
processes. So far, I have been positively surprised by Azure DevOps, and the 
convenience of running these things in the cloud is great. I use boards to 
track my work items, and pipelines to run my build processes. The repository is
hosted on GitHub, since I use that as my portfolio.

# Setting up SonarCloud

## Pre-requisites

I am going to assume that the you have set up the following accounts:

* [SonarCloud](https://sonarcloud.io/)
* [Azure DevOps](https://azure.microsoft.com/en-us/services/devops/)
* A online repository, from a service like Azure Repos, GitHub, GitLab, 
  BitBucket etc.

I am also going to assume you are vaguely familiar with Azure Pipelines. I will
try to explain the different steps added to my pipelines to the best of my 
abilities, but I might gloss over steps not related to SonarCloud. I have also
used a bit of python to glue everything together, so be prepared for that too.
Finally, as always with my articles, take it with a bit of salt. I am by no 
means an expert in any of this, so there might be better ways to do some of
these steps. If you find something that works better, by all means go for it!

## A basic setup

I started off by following the basic SonarCloud / Azure DevOps tutorial found
[here](https://docs.microsoft.com/en-us/labs/devops/sonarcloudlab/index).
However, I ran into some problems due to using C++ instead of C#. So the 
following text will be significantly inspired by the previously mentioned 
tutorial, however, it is adapted to how I got it to work with C++.

With that out of the way, let's get started, and set up our initial SonarCloud
pipeline.

### Adding the SonarCloud extension

Navigate to the [SonarCloud extension](https://marketplace.visualstudio.com/items?itemName=SonarSource.sonarcloud)
on the Visual Studio Marketplace, and install it into your project. This 
will add the required tasks into your online pipeline editor, and will save
us from a bunch of fiddling with the command line.

### Creating a new pipeline

Once you have the extension installed, there are two roads you could take.
Either you could integrate the SonarCloud steps into an existing pipeline,
possibly as a separate stage or job, or you could add an additional 
pipeline. I opted for the latter for the sake of simplicity, but do not let
that stop you from integrating it with an existing pipeline.

I created a new "Starter pipeline", using my pacman repository as the source,
and called it `sonar-cloud-pipeline.yml`.

At the time of writing, this leads to the following yaml file:

<script src="https://gist.github.com/BeardedPlatypus/a7d555bbaeb68791cf8646d812b10748.js"></script>

The current content is not particularly relevant for our use case, so I
replaced it with the basic build steps for building my pacman application:

<script src="https://gist.github.com/BeardedPlatypus/5d417571a1ff3c00b4a046366745e9a6.js"></script>

### Adding the SonarCloud tasks

SonarCloud will analyse your solution by applying various metrics to
find problems with your code. It does so by static analysis, and it
will gather the necessary data to do this while you compile your program. 

So in order to get our analysis up and running we need to take the following 
steps:

* First we need to configure SonarCloud to analyse our compile process.
* Then we need to compile our solution.
* Next we need SonarCloud to analyse the results of compiling our program.
* Finally, the results of the analysis need to be pushed to the SonarCloud server, such that they become available in the SonarCloud dashboard.

This means we will have to add the following three tasks to our pipeline:

* `SonarCloudPrepare@1`
* `SonarCloudAnalyze@1`
* `SonarCloudPublish@1`

Where `SonarCloudPrepare@1` will be placed before our build process, and the
other two will be placed after. 

### SonarCloud Prepare

The `SonarCloudPrepare@1` task will require some some setting up, which the
assistant will walk you through. First, select the "Prepare Analysis 
Configuration". We will see several options we need to configure:

* SonarCloud Service Endpoint
* Organization
* Choose the way to run the analysis
* Project Key
* Project Name
* Project Version
* Additional Properties (under advanced)

Starting off with the SonarCloud Service Endpoint. We will need to configure
the end point within our SonarCloud project, and link it to our Azure DevOps
project. First we will generate a SonarCloud token.

1. Navigate to your project on [sonarcloud.io](www.sonarcloud.io).
2. Go to "My account" (under your profile avatar in the top right corner).
3. Go to the "Security" tab, between "Profile" and "Notifications".
4. Under "Generate Tokens", add a new recognisable name. This name will not be
   used, but it does serve as a reminder for what the token was generated, such
   that you know whether you want to keep it later, or revoke it. As such, I 
   would recommend giving it a readable name, in my case it is called "PacMan 
   Azure DevOps".
5. Press the "Generate" button and copy the token. (Note, by closing this tab
   you will not be able to copy it anymore, and you will have to revoke the 
   previous tab, I would recommend either putting it in a notepad temporarily
   or leaving this tab open until you are done setting this up.)
6. With the token generated, go to your Azure DevOps project settings. (This can
   be done by selecting the little cog icon next to your Azure project name).
7. Select the "Service connections" link under "Pipelines".
8. Press "New service connection" in the top right corner.
9. Find and click the "SonarCloud" entry in the list and press "Next".
10. Add the token in the "SonarCloud Token" field and press verify.
11. Fill in a readable name in the "Service connection name", this is the name
   we will add to "SonarCloud Service Endpoint" field in our yaml file.
12. Optionally add a description.
13. Press "Verify and save". Now we are ready to add this to our yaml.

With the SonarCloud Service Endpoint configured, go ahead and add the name
you provided in step 11 within the field of "SonarCloud Service Endpoint".

In the "Organization" and "Project Key" fields, add your SonarCloud 
"Organization Key" and "Project Key" respectively. These can be found in the 
second column of your SonarCloud project dashboard.

Since I am integrating it with MSBuild, the scanner mode is selected to be
"MSBuild". "Project Name" is set to the same as my project, and for the
sake of simplicity I have set my "Project Version" to 1.0, though you can
set this to the respective version of your software, and it will be displayed
correctly in your project dashboard.

Finally press add, to add this task to your pipeline yaml. Make sure it gets
placed before the build task.

### SonarCloud Analyse and Publish

Next we will add the `SonarCloudAnalyze@1` and `SonarCloudPublish@1` tasks
after our VSBuild step. Find the "Run Code Analysis" and "Publish Quality
Gate Result" tasks in your assistant, and add them to your yaml. I did not
modify the polling time out myself, and just left it at 5 minutes.

The pipeline will now look like this:

<script src="https://gist.github.com/BeardedPlatypus/15124ef12556e4c660fccf263774d2e4.js"></script>

### Adding the build wrapper

In an ideal world, this should be enough to get SonarCloud to work. 
Unfortunately, this being a C/C++ project, we need to do some additional 
work. 

When this pipeline is run it gives an error containing the following
statement:

<script src="https://gist.github.com/BeardedPlatypus/2a9b78e380a6fbe37c42eac36ce34d57.js"></script>

In order for it to work, we will need to wrap our MSBuild process in the 
build wrapper process. The executable to do this can be obtained from 
the SonarCloud website [here](https://sonarcloud.io/static/cpp/build-wrapper-win-x86.zip).
I opted to put this directly into my repository, because I am lazy, and had
some trouble getting it to download correctly as a build step. There is nothing
stopping you from adding it as a download step instead though.

Either way, after obtaining it you should have a path to the executable. For 
the sake of convenience let's wrap this in a variable, add the following line
to your variables section of your yaml:

<script src="https://gist.github.com/BeardedPlatypus/a91cca95daf675a5f2a87c5f79774795.js"></script>

We can also add our future output directory, as mentioned in the error message,
as a variable, such that we only have to define it once:

<script src="https://gist.github.com/BeardedPlatypus/95ac4f7c86a011a40516c8cc11a9e3c2.js"></script>

With that out of the way, the next step is to modify our `VSBuild` step. Unfortunately,
our regular `VSBuild` step does not have a way to wrap it in our build wrapper, or 
I have not found a way to do this. Instead, I have changed the `VSBuild` task to 
a power shell task, and defined the command myself. It is not pretty, but it works.

The new command becomes:

<script src="https://gist.github.com/BeardedPlatypus/6a11fc85f5bf2514095b929eb22642d9.js"></script>

Let's break down what happens:

* First we define call our build wrapper executable stored in `buildWrapperExe`. 
* We set the output path of any results to the `buildWrapperOutputDir` we defined earlier.
* We call the MSBuild exe stored in `msBuildExe`, this is the regular MSBuild stored on the Azure agent.
* We make it compile our solution, and set the configuration and platform similarly to a regular `VSBuild` task.
* We also specified the option `-nologo` to skip printing the logo, such that our output log stays a bit clean.

With the build wrapper configured, the only thing we need to set is the build 
wrapper output directory within our `SonarCloudPrepare` task. This will allow
the `SonarCloudAnalyze` step to pick up on our results, and actually produce
the right results. We do this by adding the following line to the additional 
properties of SonarCloud:

<script src="https://gist.github.com/BeardedPlatypus/6a069c0f47ef664cfd45e2bfc5633f0c.js"></script>

Lastly, I have set up my SonarCloud pipeline to run once every three hours if
a commit has taken place. This will ensure I am not running my pipeline 
unnecessarily. If I do happen to need immediate feedback I tend to kick off my 
pipeline by hand anyway. In order to do this, I changed the beginning of my
pipeline yaml to the following:

<script src="https://gist.github.com/BeardedPlatypus/4c8cc75f6ac2b4476b08910e2fca6d1a.js"></script>

If you have followed along diligently and I did not make any mistake, you 
should have the following yaml code:

<script src="https://gist.github.com/BeardedPlatypus/5d884475fc5949fa384105077d24b62b.js"></script>

With all that out of the way we should have our basic SonarCloud pipeline up
and running, and we should be able to inspect the results in our SonarCloud
project dashboard. Up next, we will take a look how to connect our final metric,
code coverage.

## Adding code coverage

The last metric to set up within SonarCloud is the code coverage. I will not go
into detail about how to unit test, or why it is a good thing, enough articles
are written on these topics already, but I will add that if you are not writing
unit tests, I wholeheartedly encourage you to start doing so. If you are not
interested in the code coverage, then this is all you need to do to start using
SonarCloud, and I wish you happy code smell hunting!

Within SonarCloud we can display the code coverage metric. Once set up, 
SonarCloud will show you the overall code coverage of your solution, as well as
the coverage on newly added line, giving you a good sense of how well tested
your new code is. Furthermore, it provides a convenient interface to show which
parts of the code are untested, and let's you sort files containing uncovered 
lines in greater detail. A very useful feature in my opinion.

As mentioned previously, within the pacman project I use gtest / gmock to do my
testing in combination with the google test adapter and VSTest. When running 
the VSTest task on Azure Pipelines, you can enable measuring code coverage. 
This will add a `.coverage` file somewhere on your agent. You could use this
`.coverage` file within visual studio to show the uncovered pieces of code.
With a bit of tweaking, we can also use this data within SonarCloud.
However, the `.coverage` file is not supported out of the box,
and we will need to export it to an `.xml` file, before it will play nice.

### Producing a .coverage file

The first step into getting the code coverage set up in SonarCloud, is ensuring
the `.coverage` gets produced. As mentioned before, we can configure a VSTest
task to produce these files. Within my pacman project, I already have a CI 
pipeline in place that runs my test suite. Instead of rerunning the whole test
suite just to produce the code coverage, I figured it would be less wasteful to
reuse the data gathered during this CI pipeline. As such, the following text 
will assume you have two pipelines, one CI and one SonarCloud pipeline. The 
SonarCloud pipeline will reuse the artefacts from the CI pipeline. It should 
only be a minor inconvenience to modify the SonarCloud pipeline, to run the
VSTest itself. With that out of the way, let's set up the `.coverage` file.

We can enable the code coverage by adding the following option to our VSTest task:

<script src="https://gist.github.com/BeardedPlatypus/3a2a51925a819794c96d863dea87bc21.js"></script>

This will ensure our `.coverage` file gets produced. 

With the current iteration
of the VSTest task this file will be placed in the test results folder, located
in the TEMP folder of the Azure Agent. This location has changed in between
versions of the VSTest task already, therefor it would not necessarily be smart
to rely on an absolute path. Instead, we will configure our VSTest task to output
the `.coverage` file in a specific location. To do so, we need to create a 
`.runsettings` that specifies the output location, as shown below:

<script src="https://gist.github.com/BeardedPlatypus/7bfdab6e73c1fc3718ee20fce9a6841f.js"></script>

Where `some/Directory` is the path where the coverage files will be placed in.
In order to keep all of this as flexible as possible, I opted to generate this
`.runsettings` file during execution with a python script, which takes the path
it will generate as an argument. This way we do not need to make any 
assumptions about the file system, and instead can set this in our pipeline yaml.

The script to do this can be located [here](https://github.com/BeardedPlatypus/PacMan/blob/64ecbff3512e6205a95b02712f0ad2f73e37d978/tools/location_runsettings.py). 
I will not go over the script line by line, I hope it is mostly 
self-explanatory. But I will explain the rough idea. The script itself takes
two arguments, first the path at which we want to construct the new 
`.runsettings` file, and the path we want to output the `.coverage` file to. 
These arguments are parsed with the help of the argparse library, in the 
`parse_arguments` function. Within the `run` function, we first construct
the output directory and the parent directory of the `.runsettings` file, if
they do not exist yet. Lastly we generate the `.runsettings` file at the 
specified path with the use of the `RUNSETTINGS_TEMPLATE` in the `generate_runsettings`
function.

We can integrate this script by adding the following two tasks to our pipeline
yml anywhere before the VSTest task:

<script src="https://gist.github.com/BeardedPlatypus/3f5d0f1d88d90f542da4ca153761b70a.js"></script>

This will first set up the required Python 3 interpreter, and then execute the 
python script linked earlier. The mentioned arguments are stored in the 
variables `codeCoverageLocationRunsettings` and `testResults`. These have been
defined in the variables section of our pipeline yaml, and will be reused in
subsequent steps.

Finally, we need to modify our VSTest task slightly to make use of our generated
`.runsettings` file. As part of the `inputs` of the VSTest add the following line:

<script src="https://gist.github.com/BeardedPlatypus/7f43ff99a44380aed360163bd7106a24.js"></script>

This will ensure that the `.coverage` files are generated in `testResults`.

### Passing the .coverage files between pipelines

This section deals with interaction between the two pipelines, if you are 
running the VSTest task within your SonarCloud pipeline, you can safely skip
this section.

Now that we have our `.coverage` files generated in a known location, we can
publish them as pipeline artefacts, and download them in the SonarCloud 
pipeline. Publishing files as an artefact is done through the 
`PublishPipelineArtifact` task, or "Publish Pipeline Artifacts" in your Azure
Pipelines assistant. Go ahead and select it from the list in your assistant.
We have the following fields that we can configure:

* File or directory path
* Artifact name
* Artifact publish location

For the directory path we want to use the `testResults` variable. The name
can be anything you like, however we will use this name within the SonarCloud
pipeline, so I would not recommend going all out on it. Lastly, the Artifact 
publish location should be set to "Azure Pipelines" (unless you have a really
good reason to upload it to a fileshare, then knock yourself out).

This should lead to the following code (note, I added a simple display name
to make my build steps look nice in the log):

<script src="https://gist.github.com/BeardedPlatypus/feda76e5c449bb37a50d0a90047a3bb7.js"></script>

Next, we will download the coverage pipeline artefact within our SonarCloud
pipeline. This can be done with the `DownloadPipelineArtifact` task, or 
"Download Pipeline Artifacts" in your assistant. We have a bunch of options
to configure here.

First set the "Download artifacts produced by" to "Specific run", 
which will open up some additional options. The project should be set
to the project that contains your CI pipeline, and the "Build pipeline"
should be set to this specific pipeline. In my case these are "PacMan"
and "Build and Test PacMan" respectively. For "Build version to download"
pick "Latest". You could add a specific tag that should be present to
use to select a specific run, but I have not bothered with this.

Finally we need to set the "Artifact name", "Matching patterns", and
the "Destination directory". For "Artifact name" pick the name that
you gave the artifact in the previous step (`coverage` in the snipped above).
Since we are only interested in the `.coverage` file, I have specified 
my "Matching patterns" to `**/*.coverage`, this will look recursively
in all the folders of my artefact for any file ending with `.coverage`.
As a final step, download the files somewhere on your agent. In my case
I put it in a variable called `coverageDownloadLocation`. This should 
put a task in your yaml comparable to:

<script src="https://gist.github.com/BeardedPlatypus/322a9de24d68c20e9f12cf1d2451ca71.js"></script>

Next up, we will convert the `.coverage` file to `.xml` and feed it into
the SonarCloud analysis.

### Converting the .coverage file

Converting the `.coverage` files to `.xml` is a reasonably easy process, once
you have figured out how to do it. The path to figuring it out though, is
painful and filled with perils, so hopefully you can learn from my mistakes
and it will be a breeze for you. First and foremost let me state, that if you
want to use paths with spaces in your python and command line scripts, make sure
you use double quotes, `"`, and not single quotes, `'`. Yes ... it took me longer
than I would like to admit to figure that out. With that out of the way, let's
look at how we can convert `.coverage` files in to something usable by SonarCloud.

If my understanding is correct, the `.coverage` is basically a binary blob of
code coverage information, which can be used by Visual Studio to give you an
indication of your code coverage. Unfortunately, it does not work out of the 
box for SonarCloud. This means we need to convert it to an `.xml` file which will be
usable. There are various ways of doing this, but the easiest I have found is 
to use the `codecoverage.exe` provided with Visual Studio Enterprise, i.e. the
Visual Studio version that is installed on the Azure agents.

When we run the following command, it will convert the `.coverage` into an `.xml`
file:

<script src="https://gist.github.com/BeardedPlatypus/8970b5d2b2588d0df38bcc3f9ad097d2.js"></script>

Where input and output can basically be any name you want it to be. 

Because I did not want to hard code the location of the `codecoverage.exe`,
and I do not know the `.coverage` file names without running actually running
the pipeline, I ended up writing another python script to dynamically resolve
these things. The script can be found [here](https://github.com/BeardedPlatypus/PacMan/blob/3d723576abfbf06da38007984b37b9696efd89a6/tools/convert_coverage.py).
Again, I will not go over the script line by line, but I will explain the rough
set up. The script takes one argument, the folder in which the `.coverage` files
are located. It then resolves the `codecoverage.exe` path by searching for it in
the Visual Studio folder of the agent. It will then copy the `.coverage` files from 
the specified path to the current working directory. Lastly, it will convert the
recently copied files from `.coverage` to `.xml` using the `codecoverage.exe`.
Once done, the current working directory should contain the relevant `.xml` files
(and the original `.coverage` files). 

This script is ran in the same way as before, by adding two python tasks (after
the download pipeline artifacts task):

<script src="https://gist.github.com/BeardedPlatypus/baa6114cb11198c6ba6f5dc4f8b3ca74.js"></script>

We set up the argument and the working directory with the appropriate variables
again defined in the variable section of the yaml. If you are interested in 
seeing the output, you could publish the defined working directory as an artefact,
such that they can be downloaded and inspected.

### Modifying the SonarCloud tasks

The final step to wrap up our SonarCloud setup, is to use the newly generated
`.xml` files within our analyses. For this we need to add one more line to our
`extraProperties`:

<script src="https://gist.github.com/BeardedPlatypus/56f930fe91c676045407cf2d582eb8b2.js"></script>

Where `coverageFiles` is defined as:

<script src="https://gist.github.com/BeardedPlatypus/0ceefdd5f9319e395ef41c608a5f9ef4.js"></script>

Which says all the `.xml` files within the location where we generated our `.xml`
files. This should ensure that SonarCloud takes into account the code coverage,
and you can keep an eye on maintaining that 80%+ metric!

# Conclusion

This guide turned out to be slightly longer than I originally intended, but if
you followed along, and I did not make any mistakes, you should have a pipeline
looking similar to [my SonarCloud pipeline](https://github.com/BeardedPlatypus/PacMan/blob/7127d4b26988f3442b811a2225583e775bc7b0d9/sonarcloud-pipeline.yml). Which should provide you with a dashboard similar to
[my pacman project](https://sonarcloud.io/dashboard?id=BeardedPlatypus_PacMan).

I hope this made the process of setting up SonarCloud for a C++ project slightly
easier for you. Thank you for reading! And if you have any comments, suggestions, 
or questions, let me know!
