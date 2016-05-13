---
layout:     post
title:      Running Atlasboard on an Azure Web App
date:       2016-05-13 01:30:00
categories: atlasboard, azure, javascript
---

If you are a software developer like me, you are probably excited about graphs and stats from 
systems that are relevant for your workflow. If this data is available on a live updated 
big screen on the wall, it´s even better. [Atlasboard](http://atlasboard.bitbucket.org) is
one of the tools that can easily get you a nice looking status board.

Atlasboard is an open source nodejs based application created by Atlasian. You can read more
about it [here](http://atlasboard.bitbucket.org/).

Since I´m a .net developer and have most of my apps on Azure, I wanted to host Atlasboard in an
Azure Web App. Here is a walk through of how I got it up and running.


Start by creating a new web app through the [Azure Portal](https://portal.azure.com). You could also do this from the 
command line using the Azure CLI, but I´m going to use the Azure Portal in this walk through.

![Screenshot of Azure Portal]({{ site.url }}/assets/posts/atlasboard/1.png)

Make sure you have [Node](https://nodejs.org/en/) installed.

Run ```npm install -g atlasboard```. This will install atlasboard as a global package on your local machine.

Then use the new atlasboard command you just installed to create a new dashboard by issuing the command

```atlasboard new mydashboard```

This will create a new directory structure with your new dashboard. 

{% highlight %}

-rw-r--r--  1 henning  staff  139 May 12 22:37 README.md
drwxr-xr-x  4 henning  staff  136 May 12 22:37 assets
drwxr-xr-x  6 henning  staff  204 May 12 22:37 config
-rw-r--r--  1 henning  staff  192 May 12 22:37 globalAuth.json.sample
-rw-r--r--  1 henning  staff  502 May 12 22:37 package.json
drwxr-xr-x  4 henning  staff  136 May 12 22:37 packages
-rw-r--r--  1 henning  staff  580 May 12 22:37 start.js
drwxr-xr-x  3 henning  staff  102 May 12 22:37 themes
{% endhighlight %}

Then we need to add the atlasboard dependency in package.json like this:


{% highlight json %}
  "dependencies": {
    "atlasboard": "^1.1.3"
  }
{% endhighlight %}


cd into mydashboard and run npm install to install the required packages.

You should now be able to run ´´´atlasboard start 3333``` to start atlasboard. Open http://localhost:3333 in a browser to make sure it works.

Before you can deploy to Azure, there are a couple of adjustments you need to do.

1. Change the port number variable from ATLASTBOARD_PORT to PORT as shown below.

Before:
{% highlight javascript %}
atlasboard({port: process.env.ATLASBOARD_PORT || 3000, install: true}, function (err) { 
{% endhighlight %}

After:
{% highlight javascript %}
atlasboard({port: process.env.PORT || 3000, install: true}, function (err) {
{% endhighlight %}



2. Change the required npm version in package.json

Before:
{% highlight javascript %}
  "engines": {
    "npm": "~2.0.0",
    "node": ">=0.10"
  },
{% endhighlight %}

After:
{% highlight javascript %}
  "engines": {
    "npm": ">2.0.0",
    "node": ">=0.10"
  },
{% endhighlight %}
 


We are now ready to deploy. I´ll choose to deploy directly from a local git repository but you 
could deploy from github, vsts etc. To enable git deployments to your Azure web app,
go to the Azure portal again, click "Deployment source" and choose "Local Git repository" as
the deployment source as shown in the screenshot below.

![Screenshot of Deployment source]({{ site.url }}/assets/posts/atlasboard/2.png)


Then open "Deployment credentials" and set a username and password. This will be the credentials
you use when adding Azure as a remote to your local git repository.

The next thing you need to do is to initialize the mydashboard directory as a git repository.
First initialize a local git repository in your mydashboard directory by typing git init.

Initialized empty Git repository in /Users/henning/Projects/henningst-atlastest/mydashboard/.git/

Now add the git url of your web app as a remote in order to push to azure.

git remote add azure https://henningst-atlasboard@henningst-atlasboard.scm.azurewebsites.net:443/henningst-atlasboard.git

Add and commit all files and push to azure.
git add .;git commit -m "Initial commit";git push azure master

You should see:

remote: Deployment successful.



After the site had been deployed and, the Kudu system in Azure makes sure the atlasboard npm package is installed. However, there is a ..... blabla. You need to modify one of the installed js files manually.

This can be done through the Kudu Console that you find using the following URL
https://[your-webapp-name].*scm*.azurewebsites.net/DebugConsole (or click tools->kudu as shown in the screenshot below.

Locate the following file and edit it.
D:\home\site\wwwroot\node_modules\atlasboard\lib\package-dependency-manager.js

Around line 92:

  var npmCommand = isWindows ? "npm.cmd" : "npm";

  executeCommand(npmCommand, ["install", "--production", pathPackageJson], function(err, code){


  var npmCommand = isWindows ? "D:\\Program Files (x86)\\npm\\3.5.1\\npm.cmd" : "npm";

  executeCommand(npmCommand, ["install", pathPackageJson], function(err, code){
  
  
If you access the web app url, you should now see the demo dashboard up and running!


## Trouble shooting

Add the following two lines to iisnode.yml. This will enable logging for iisnode and you´ll see log files starting to show up inside wwwroot/iisnode.

loggingEnabled: true
logDirectory: iisnode
