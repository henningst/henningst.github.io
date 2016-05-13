---
layout:     post
title:      Running Atlasboard on an Azure Web App
date:       2016-05-13 00:30:00
summary:    Walk-through of how you can get Atlasboard up and running in an Azure Web App.
categories: atlasboard, azure, javascript
---

If you are a software developer like me, you are probably excited about graphs and stats from 
systems that are relevant for your workflow. If this data is available on a live updated 
big screen on the wall, it´s even better. [Atlasboard](http://atlasboard.bitbucket.org) is
one of the tools that can easily get you a nice looking status board.

Atlasboard is an open source nodejs based application created by [http://www.atlassian.com](Atlassian). 

Since I´m a .net developer and have most of my apps on Azure, I wanted to host Atlasboard in an
Azure Web App. Here is a walk through of how I got it up and running.

Start by creating a new web app through the [Azure Portal](https://portal.azure.com). You could also do this from the 
command line using the Azure CLI, but I´m going to use the Azure Portal in this walk through.

![Screenshot of Azure Portal]({{ site.url }}/assets/img/posts/atlasboard/1.png)

Make sure you have [Node](https://nodejs.org/en/) installed.

Run ```npm install -g atlasboard```. This will install atlasboard as a global package on your local machine.

Then use the new atlasboard command you just installed to create a new dashboard by issuing the command

```atlasboard new mydashboard```

This will create a new directory structure with your new dashboard. 

{% highlight bash %}

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

![Screenshot of Deployment source]({{ site.url }}/assets/img/posts/atlasboard/2.png)


Then open "Deployment credentials" and set a username and password. This will be the credentials
you use when adding Azure as a remote to your local git repository.

The next thing you need to do is to initialize the mydashboard directory as a git repository by
doing a ```git init```. Make sure you do this inside the root directory of the dashboard you have created.

Now add the git url of your web app as a remote in order to push to Azure.

```git remote add azure https://[username]@[your-webapp-name].scm.azurewebsites.net:443/[your-webapp-name].git```

You find the Git URL for your web app in the Azure portal as shown in the screenshot below.

![Screenshot of Azure settings with Git URL]({{ site.url }}/assets/img/posts/atlasboard/3.png)

Add and commit all files and push to Azure.
```git add .;git commit -m "Initial commit";git push azure master```

You should now see all the files being pushed and you will also see some output from Kudu that takes care
of installing your node app. If everything goes as expected, you should see ```remote: Deployment successful.```
as one of the last lines of the output.

## Tweak package-dependency-manager.js
Your dashboard is now installed in Azure, but there is a small hack you need to do in order to
get this working. In the Atlasboard dependency manager, you need to change the path to the npm command
and modify the command that is issued when installing packages.

This can be done through the Kudu Console which you can find using the following URL
https://[your-webapp-name].*scm*.azurewebsites.net/DebugConsole. There is also a link to the
Kudu dashboard from the Azure Portal (Tools --> Kudu --> Go) as shown in the screenshot below. 

![Screenshot of Azure settings with Git URL]({{ site.url }}/assets/img/posts/atlasboard/4.png)


Locate the following file and edit it.
```D:\home\site\wwwroot\node_modules\atlasboard\lib\package-dependency-manager.js```

Around line 92:

Before:
{% highlight javascript %}
  var npmCommand = isWindows ? "npm.cmd" : "npm";

  executeCommand(npmCommand, ["install", "--production", pathPackageJson], function(err, code){
{% endhighlight %}

After:
{% highlight javascript %}
  var npmCommand = isWindows ? "D:\\Program Files (x86)\\npm\\3.5.1\\npm.cmd" : "npm";

  executeCommand(npmCommand, ["install", pathPackageJson], function(err, code){
{% endhighlight %}
  
  
If you access the web app url, you should now see the demo dashboard up and running!

![Screenshot of a running Atlasboard]({{ site.url }}/assets/img/posts/atlasboard/5.png)

## Thanks

Thanks [@garyliu](https://twitter.com/garyliu) to for helping me get this up and running by answering
my [stackoverflow question](http://stackoverflow.com/a/37109868/5795)


## Trouble shooting
If you still run into problems, it might be useful to add the following lines to the iisnode.yml file
in wwwroot where your webapp is installed.

{% highlight javascript %}
loggingEnabled: true
logDirectory: iisnode
{% endhighlight %}


This will enable logging for iisnode and you´ll see log files starting to show up inside wwwroot/iisnode.

