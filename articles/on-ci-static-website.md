title: Setting up Continuous Delivery for your static website.
date: 2019-07-21

In this article we will take a short look how to set up continuous delivery for
a simple static website (my own, woo!) with Azure DevOps. Once set up, any 
commit to both content and the theme should trigger the right builds that end up
building and pushing the changes to a github pages website. For my own website
I use pelican, however, the content in this article should be easily adjustable
to any static website!


# Motivation

Lately, I have been wanting to start writing and posting some blog articles 
again. Having an active blog has been something I want for quite a while (I 
think my original github pages was created afew years ago), however I never took
the time to properly finish any of the articles, as polishing my own personal 
projects has never been a strong suit of mine. 

These past few weeks I have been working on some small personal projects that I
want to document in some fashion, such that I can understand what I did in the
future. I figured this would be an excellent opportunity to create some blog 
articles that are worth sharing. 

When I looked at this dusty bit of webspace, I noticed however, that it had not
aged as well as I had hoped. So as would be expected from my procrastinating 
nature, I decided to first dust it off, give it a lick of paint, before doing 
the thing I actually want to do, which is write the blog articles.

As part of the dusting off, I figured I would automate some steps that felt 
like a drag. Because the website is a generated static website, whenever I made 
a change to either the the theme or the content, I would need to run all the 
compilation steps by hand, and then push the results to my github pages 
repository. Lazy as I am, this cannot be of course. 

As I have been working with Azure DevOps lately in some other personal projects, 
I figured it would be a good choice to automate the compilation with. These 
build steps can then be executed as part of a build pipeline running on the 
servers of microsoft. And because of that idea, you can now read this article!

# Project Configuration

The main configuration of my website can be found in 
[this repository](https://github.com/BeardedPlatypus/personal-website-config)
As mentioned before, the website is created with pelican, a static website 
generator written in python. There are a bunch of alternatives on the market 
(Jekyll comes to mind but also newer generators like Hugo), however I picked 
pelican because of its python support, a language I am already quite comfortable
with. As much as I enjoy learning new languages, in this case I wanted something
I could get up and running without struggling too much (of course being me, I 
did struggle with it, but that's besides the point). 

Besides pelican I have used Gulp to compile my own crafted theme, and do a little
bit of optimisation. The examples within this blog post will thus be focused on
these technologies. However, it should be rather trivial to adapt the examples 
to any other static website generator. In this section I will quickly go over 
the project structure,and how the technologies are set up. For further detail 
I would refer you to the documentation of the specific tools, as the focus of
the rest of the article is on how to set up the Azure DevOps pipelines.

## Folder Structure

The static website basically consists of three parts, the theme, which can be
found in [this repository](https://github.com/BeardedPlatypus/rubber-squid), 
the content, which can be found in 
[this repository](https://github.com/BeardedPlatypus/personal-website-content)
and lastly the configuration files, which can be found in 
[this repository](https://github.com/BeardedPlatypus/personal-website-config). 

After completing a full run of the compilation of the website we will have 
the following (simplified) directory structure:

```
Base                  # config git repository
├───content           # content git repository
│   ├───articles
│   └───pages
├───plugins           # plugins used by pelican
├───production        # output folder of pelican
├───theme
│   └───rubber-squid
│       ├───static
│       └───templates
└───website_dist      # output of gulp postPelican
```

As you can see, the content folder is checked out as separate repository in the
config folder, and the theme is checked out in the theme folder. I could have 
set this up with the help of git submodules, however I have not done this yet,
and for the current purposes, this set up suffices.

## Pelican

Pelican is configured through the 
[pelican.conf](https://github.com/BeardedPlatypus/personal-website-config/blob/master/pelicanconf.py)
and the 
[publish.conf](https://github.com/BeardedPlatypus/personal-website-config/blob/master/publishconf.py).
Within these files things like the theme, names etc are specified. In addition
to the default pelican, I also use the summary plugin, which helps creating the
summaries for the blog index page. I have modified this slightly to remove any
links from within the summaries.

When pelican is properly installed, the static website can be generated through
the following command: 

```
pelican content -s publishconf.py
```

This will create all the html pages based on your content, theme and 
configuration in the `production` folder. The contents of this folder
could be uploaded immediately to a github pages repository, and it 
would be available for viewing. However we can do a little bit better and 
optimise the result of pelican with some javascript magic.

## Gulp

When I created the previous incarnation of this website, I did not know anything
about web optimisation, since then I have learned some new tricks and figured 
I wanted to incorporate them. Building my website now has a pre and post step
besides the pelican compile step.

Before we can build the website, we first need to compile the theme, or more 
specifically, the sass files, such that we have some css that pelican can use.
As mentioned before, once the compilation is done, we can then further optimise
the output a bit by calling some other javascript magic.

Both of these steps are managed by [Gulp](https://gulpjs.com/).
I set up a simple  workflow, based upon 
[this tutorial](https://css-tricks.com/gulp-for-beginners/) to do so.

The pre step at the moment of writing merely compiles the sass files through the
following gulp command:

```
gulp prePelican
```

The post pelican step collects all css files, concatenates them, and minimises them,
and puts them in a `website_dist` folder. It also copies over any of the html, 
js, and feed files. I currently have not set up the minimisation of the js file, 
as I already did this in a separate step, when I created my own js. However in the
future there is a good chance I will add this. This step can be executed as follows:

```
gulp postPelican
```

The steps themself are set up in the `gulp.js` file of the 
[config repository](https://github.com/BeardedPlatypus/personal-website-config/blob/master/gulpfile.js).

Keep in mind that I am nowhere near experienced with any of this, so take this 
set up with a grain of salt, and do your own research in order to find a set up 
that works well for you!

# Compile the website on Azure DevOps

With the extremely short primer out of the way, let's take a look at the actual
build configuration. In essence, we want to do 4 steps:

* compile the theme
* compile pelican
* optimise the pelican output
* push it all to the github pages

in the following section we will take a look at each of these steps, if however
you just want to see the configuration, it can be found 
[here](https://github.com/BeardedPlatypus/personal-website-config/blob/master/azure-pipelines.yml).

For the sake of brevity, I am going to assume you already have a project set up, 
and can know where to find the add build pipeline button. If you do not know this,
I recommend going over the documentation of Azure DevOps, it should not take long to
set up.

## Setting up the Pelican Components

In order to set up our pipeline, we need to make use of python and javascript. We will
start with python, and then move on to the nodejs component.

The main build line should be based in our config repository, as this one will hold all
the information to actually build the website. As such, I selected GitHub as my source 
of my code, and then the config repository. Since we want to use python for our pelican
application build, I started out with the python package configuration as base. 

We only need a single python version, as we only build our website once. Furthermore,
at the time of writing I did not include any python testing, so I removed this step to.
This leaves us with the following yml code:

```yml
# Python package
# Create and test a Python package on multiple Python versions.
# Add steps that analyze code, save the dist with the build record, publish to a PyPI-compatible index, and more:
# https://docs.microsoft.com/azure/devops/pipelines/languages/python

trigger:
- master

pool:
  vmImage: 'ubuntu-latest'
strategy:
  matrix:
    Python37:
      python.version: '3.7'

steps:
- task: UsePythonVersion@0
  inputs:
    versionSpec: '$(python.version)'
  displayName: 'Use Python $(python.version)'

- script: |
    python -m pip install --upgrade pip
    pip install -r requirements.txt
  displayName: 'Install dependencies'
```

First it defines a trigger, this means that whenever a commit is made on the 
branches listed under it, this specific build will trigger. Since we only want
to actually build and push or website when we make a definitive commit, we can
leave this on master.

Next some strategies are defined. If we had an actual python package to test, 
we might want to do that for different versions. However, we only need one 
version of python, so we can remove all but the 3.7 version, as this is the 
latest.

In the following step it gets the specific python version, and then 
installs our requirements.txt file. The requirements file specifies which packages
need to be installed. For pelican we need both the pelican package and the markdown
package, as such it looks liks this:

```
Markdown==3.1.1
pelican==4.0.1
```

Once these steps have executed, we have our python installation ready to use, and
we can add the pelican compile step.

```yml
- script: |
    pelican content -s publishconf.py
  displayName: 'Pelican Compile'
```

As you can see, it merely executes our pelican command, as we defined it the previous
section. The `displayName` will be shown in the summary of Azure DevOps, and makes
debugging a bit easier, so I try to give it a relevant name, but you could always
change or even remove it.

We can add this part right after the python bit and execute it. Unfortunately. This
code does not work yet, as we do not have our data to act on. Before we set this up
though, let's add the javascript bit to our code.

## Setting up the nodejs / npm components

If we instead had started with Node.js pipeline, our template would have looked like 
this:

```yml
# Node.js
# Build a general Node.js project with npm.
# Add steps that analyze code, save build artifacts, deploy, and more:
# https://docs.microsoft.com/azure/devops/pipelines/languages/javascript

trigger:
- master

pool:
  vmImage: 'ubuntu-latest'

steps:
- task: NodeTool@0
  inputs:
    versionSpec: '10.x'
  displayName: 'Install Node.js'

- script: |
    npm install
    npm run build
  displayName: 'npm install and build'

```

It uses the same pool and trigger as our python script, and contains the steps to
install Node.js and npm. Therefore we can just copy these install steps to the 
configuration we are setting up.

We can remove the npm run build step, as we are not building an actual Node.js 
application, we merely want it to install gulp. Once we call the npm
install step, all the packages from our `package.json` are installed. For this
website this is basically gulp, and some supporting plugins, see 
[here](https://github.com/BeardedPlatypus/personal-website-config/blob/master/package.json).
After running the install, we can use gulp to compile our team and optimise the
output of pelican.

Since we defined all our pre-pelican steps in one command, the step in our configuration
becomes:

```yml
- script: |
    gulp prePelican
  displayName: 'Pre-Pelican Compile'
```

and the post pelican step becomes:

``` yml
- script: |
    gulp postPelican
  displayName: 'Post-Pelican Compile'
```

We put these commands before and after our pelican step:

```yml
- task: UsePythonVersion@0
  inputs:
    versionSpec: '$(python.version)'
  displayName: 'Use Python $(python.version)'

- script: |
    python -m pip install --upgrade pip
    pip install -r requirements.txt
  displayName: 'Install python dependencies'

- task: NodeTool@0
  inputs:
    versionSpec: '10.x'
  displayName: 'Install Node.js'

- script: |
    npm install
  displayName: 'npm install'

- script: |
    gulp prePelican
  displayName: 'Pre-Pelican Compile'

- script: |
    pelican content -s publishconf.py
  displayName: 'Pelican Compile'

- script: |
    gulp postPelican
  displayName: 'Post-Pelican Compile'
```

With that set up, we are basically all ready to execute our compilation process.
Next up, getting the data.

## Checking out the additional repositories

In order to get our data from github, we basically need to clone our content and
theme repositories. Since I was having some troubles with the script task, I opted
to use the powershell task. This task basically gives access to a powershell instance
in your configuration, and allows you to run a variety of commands. 

Since git comes pre-installed, we do not have to worry about it, and can just use
git commands in our powershell task.

There might be reasons why you want to keep your theme and content into private
repositories, in order to clone these repositories you will need to authenticate
yourself. Ideally, you would use some token based way to authenticate yourself,
however I have not come around to do so in my own pipeline, so instead we can alse  use 
basic authentication for this within github. In order to do so we need to specify 
our URL as follows:

```
https:\\<account_name:account_pw@github.com/<repository account name>/<repository name>.git
```

We do not want to leak this information in our build logs. Therefore we are going 
to use private variables, 
[as microsoft recommends](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=yaml%2Cbatch).

Thus our clone steps then become:

```yml
- powershell: |
    $url = "github.com/BeardedPlatypus/rubber-squid.git"
    $url = ":$env:GITHUB_PW@$url"
    $url = "$env:GITHUB_ACCOUNT$url"
    $url = "https://$url"
    git clone $url "theme/rubber-squid"
  displayName: 'Check out - Rubber Squid Theme'
  env:
    GITHUB_ACCOUNT: $(github.profile)
    GITHUB_PW: $(github.password)

- powershell: |
    $url = "github.com/BeardedPlatypus/personal-website-content.git"
    $url = ":$env:GITHUB_PW@$url"
    $url = "$env:GITHUB_ACCOUNT$url"
    $url = "https://$url"
    git clone $url content
  displayName: 'Check out - Blog Content'
  env:
    GITHUB_ACCOUNT: $(github.profile)
    GITHUB_PW: $(github.password)
```

The first steps are done to build our URL as described above. I had 
some trouble getting this to work too be honest, and this ended up
working correctly, so I decided to stick with it.

Once we have build our URL, we can clone our repositories. Because
pelican expects them in a specific place relative to our working 
directory, we add the folder we want them in.

I have added these steps before I even install python or Node.js.
The chance that these steps fail as opposed to the install steps
is greater, and we rather fail earlier than later.

## Getting some artifacts

Now that we have all our compilation steps set up, once we run
our build confiration, we should have a nice `website_dist` folder,
containing our whole website. Before we push this to our github 
pages, let's add some steps to publish this as an artifact as well.

```yml
- task: CopyFiles@2
  inputs:
    contents: 'website_dist/**'
    targetFolder: $(Build.ArtifactStagingDirectory)
  displayName: 'Artifacts - Copy'

- task: PublishBuildArtifacts@1
  inputs:
    pathToPublish: $(Build.ArtifactStagingDirectory)
    artifactName: websiteContents
  displayName: 'Artifacts - Publish'
```

This will output our `website_dist` for us to inspect, and ensure
the website was generated correctly. We could further extend this
to also publish the output of pelican, or the first gulp step 
for example. This could help with debugging.

## Pushing the content to github pages

In order to push our content to our github pages repository, we
need to do the following things:

* Clear our repository
* Add the output of our postPelican step
* Commit the changes
* Push the content

In order to clear out the content of our current github pages
repository, we can create an empty check out of our repo, in
the `website_dist` folder. This can be achieved with the 
following step:

```yml
- powershell: |
    git clone --no-checkout "https://github.com/BeardedPlatypus/BeardedPlatypus.github.io" website_dist
  displayName: 'Check out - BeardedPlatypus.io'
```

The `--no-checkout` flag will ensure we do not check out any files.
If we were to commit this as is, it would clear our whole repository.
We want to put this step before our `postPelican` step, such that the
folder is still empty (otherwise git will complain).

Now, once we have passed the `postPelican` step, we can actually 
add all the files generated, commit, and push them. The step to
do this looks as follows:

```yml
- powershell: |
    git config user.email "$env:USER_EMAIL"
    git config user.name "$env:USER_NAME"
    
    git add *
    git commit -m "Automated update: $(Build.BuildNumber)"
    
    $url = "github.com/BeardedPlatypus/BeardedPlatypus.github.io.git"
    $url = ":$env:GITHUB_PW@$url"
    $url = "$env:GITHUB_ACCOUNT$url"
    $url = "https://$url"
    
    git push $url master
  displayName: 'Update BeardedPlatypus.io'
  workingDirectory: website_dist
  env:
    GITHUB_ACCOUNT: $(github.profile)
    GITHUB_PW: $(github.password)
    USER_NAME: $(user.name)
    USER_EMAIL: $(user.email)
```

We will execute this step in the `website_dist` folder, as such
the `workingDirectory` property of this step is pointing to this
folder. This will make it possible to refer just to the content
of this folder, and use the `--no-checkout` local repository we 
just cloned.

We start by setting up our `user.email` and `user.name` in order to be 
able to push our content to the repository in the final step. 
Next, we add and commit all the changes currently existing in the 
folder. Because the empty folder should already have been staged due
to the `--no-checkout` flag, this should leave us with the files as 
they are in `website_dist`. Furthermore, if there are no changes, 
git will just say everything is up to date, and finish up everything. 

After this, we again build our url, and push to the master branch of
the github pages repository. And with that we have our automatic 
deployment of our static website. 

The full `azure-pipelines.yml` can be found 
[here](https://github.com/BeardedPlatypus/personal-website-config/blob/master/azure-pipelines.yml).

## Triggering the build on content and theme changes

Unfortunately, we currently only trigger the build automatically when
we make a change to the configuration repository. Ideally, this repository
would change little, and most of the changes would be either to
the content or the theme. Thus, whenever we make a change to either 
of these repositories, we would need to do a manual run, which really
is not that much better than doing it by hand on your local machine.

We can do better! As far as I know, we cannot directly add triggers
that go off once we make changes to these repositories, as one build
only works with one base repository. However, we can make our current
build trigger upon the completion of other builds. Therefor, we will
add some really simple build pipelines for these two repositories:

```yml
trigger:
- master

pool:
  vmImage: 'ubuntu-latest'

steps:
- script: |
    echo "Theme has been triggered."
  displayName: 'Theme'
```

You could in theory extend these with some sanity or sanitary checks,
such that you do not kick off builds if you know something is wrong,
for now however I have set them up to merely do a simple echo and 
wrap up.

Once we have set up these pipelines, you can add the completion of
these pipelines as triggers under the trigger settings of our
main build pipeline. Once that is done, our main build should 
trigger automatically. You can try it by running our newly added
pipeline once, and then check if our main gets triggered subsequently.

# Conclusion

In this article, we set up a simple Azure DevOps build to compile a static
website. We have gone over how to set up the python task to execute pelican,
and how to further optimise the results with the help of gulp. Finally, we
have set it up in such a way that changes to either the theme or content
repositories will ensure the website gets automatically updated and pushed
to the github pages repository.

We could further extend this pipeline by adding additional optimisation. 
Furthermore, the build pipelines of both the theme and content could be
extended with additional checks, to ensure we only push when we can create
a valid website. For the time being though I am quite happy with the result.

Thanks for reading, and I hope to see you soon again in another article!

