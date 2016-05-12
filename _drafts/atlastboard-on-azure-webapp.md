Testing atlasboard on Azure webapps

- Created a new repo on vsts
- Installed atlasboard
- Moved the site to the root directory of the repo
- Changed the node pinned version (see deployment log)
  - logfiles in iisnode dir
- Enabled iisnode logging by changing the nodeiis.yml file
- problems with node modules in repo


Title: How to run AtlasBoard on an Azure Web App

Start by creating a new web app through the Azure Portal. You could also do this from the command line using the Azure CLI.

Then head over to http://atlasboard.bitbucket.org/ and follow the steps to get started in 30 seconds.

Make sure you have Node installed. You can check this by typing ```node -v``` in the console.

run npm install -g atlasboard. This will install atlasboard as a global package on your local machine.

Then use the new atlasboard command you just installed to create a new dashboard by issuing the command

atlasboard new mydashboard

this will create a new directory structure with your new dashboard. 

-rw-r--r--  1 henning  staff  139 May 12 22:37 README.md
drwxr-xr-x  4 henning  staff  136 May 12 22:37 assets
drwxr-xr-x  6 henning  staff  204 May 12 22:37 config
-rw-r--r--  1 henning  staff  192 May 12 22:37 globalAuth.json.sample
-rw-r--r--  1 henning  staff  502 May 12 22:37 package.json
drwxr-xr-x  4 henning  staff  136 May 12 22:37 packages
-rw-r--r--  1 henning  staff  580 May 12 22:37 start.js
drwxr-xr-x  3 henning  staff  102 May 12 22:37 themes

Then we need to add the atlasboard dependency in package.json by adding

  "dependencies": {
    "atlasboard": "^1.1.3"
  }


!!!! Should we do some steps here first?


cd into mydashboard and run npm install to install the required packages.

You should now be able to run ´´´atlasboard start 3333``` to start atlasboard. Open http://localhost:3333 in a browser to make sure it works.

Before you can deploy to Azure, there are a couple of adjustments you need to do.

1. Change the port number variable from ATLASTBOARD_PORT to PORT as shown below.

atlasboard({port: process.env.ATLASBOARD_PORT || 3000, install: true}, function (err) { 
atlasboard({port: process.env.PORT || 3000, install: true}, function (err) {

2. Change the required npm version in package.json

  "engines": {
    "npm": "~2.0.0",
    "node": ">=0.10"
  },

  "engines": {
    "npm": ">2.0.0",
    "node": ">=0.10"
  },
 


We are now ready to deploy. I´ll choose to deploy directly from a local git repository but you could deploy from github, vsts etc. Select local git repository and click ok (see screenshot).

Then open deployment credentials and set a username and password.

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
