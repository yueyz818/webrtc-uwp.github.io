# Building in Azure DevOps

These are the tasks that are to be run in the pipeline. If the field within the task is not referenced below, defer to the default option, or leave as blank

> Note: The below steps are currently not possible on the free version of DevOps due to the 10Gb disk space restriction.

1. Pipeline
Ensure the Agent pool value is set to Hosted VS2017 (unless a custom agent is being used, which is recommended, if not required, as the builds require a significant amount of space, and the amount of time required for a build to complete decreases when using a custom VM).
2. Get Sources
Set the Source to Azure Repos Git
Connect the repository to webrtc-uwp-sdk.
Select the default branch to the one that is needed scheduled builds (typically the master branch).
Ensure Checkout submodules is selected.
3. Variable
Name: config.actions  Value: prepare build backup
Name: config.cleanIDL Value: true
Name: config.configurations Value: debug,release
Name: config.cpus Value: x86 x64
Name: config.packNuget Value: true
Name: config.platforms Value: winuwp,win
Name: config.targets Value: webrtc
4. Options
Build job time out in minutes is set to 0.
5. Setup pipeline
There are two ways to handle this task, run on the hosted agent or setup a VM.
5.1. Setup on Hosted Agent
5.1.1 Chocolatey
This task utilizes Windows' Chocolately Package Manager to install Windows SDK version 10, along with the Windows Debugger Tool
Download the SDK 1803
Add the extension to the secure file as follow:
On the left-side of the project screen, under Pipelines, click on Library -> Secure files -> "+Secure File"
Upload the downloaded SDK 1803
Options:
Command: install
The id of the package(s) that are to be installed: windows-sdk-10-version-1803-windbg
5.2 Setup using a Custom VM as Build Agent(s)
Create a custom VM following the instructions in this repo for building and deploying a VS2017 + Server 2016 image.
The process takes ~8 hours to complete, so you must either leave your system on for 8 hours, or use a different VM to generate this one.
Follow these instructions to set your VM up as a build agent.

5.2.1 Install Chocolatey on the VM
Log into the VM and install Chocolatey

6. Use Python 2.7.14
Use this Python version. Note: the version needs to be specific in this case. Following are the settings required for this agent:
Version Spec: 2.7.14
Ensure that 'Add To Path' is checked
Advanced:
Architecture: x64
7. Build UWP Release Version
Create a Python Script agent for this section. There should be two different release agents, one for x86 and one for x64. Settings are as follow:
Script path:
$(Build.SourcesDirectory)\scripts\run.py
Arguments:
-a prepare build -t webrtc -p winuwp --cpus [x86|x64] -c $(config.configurations)
working directory:
$(Build.SourcesDirectory)
Custom condition :
eq(variables['config.configurations'], 'release')

8. Build UWP Debug Version
Create a Python Script agent for this section. There should be two different debug agents, one for x86 and one for x64. Settings are as follow:
Script path: 
$(Build.SourcesDirectory)\scripts\run.py
Arguments:
-a prepare build -t webrtc -p winuwp --cpus [x86|x64] -c debug
working directory:
$(Build.SourcesDirectory)
Custom condition :
eq(variables['config.configurations'], 'debug')
9. Clean Up Intermediates (Hosted Agents Only)
After each set of build and release agents we need to clean up the memory. For this section, use command line agent with the following setting:
Script:
 echo Removing Intermediates
if exist "%Build_SourcesDirectory%\webrtc\windows\solutions\Build\Intermediate" rmdir %Build_SourcesDirectory%\webrtc\windows\solutions\Build\Intermediate /S /Q
if exist "%Build_SourcesDirectory%\webrtc\xplatform\webrtc\out" rmdir %Build_SourcesDirectory%\webrtc\xplatform\webrtc\out /S /Q
10. Use NugetUWPCI.py to create Nuget Package
This file is located in the Templates folder in the project.  It is generated by the default.py template and is customized for this specific project to create Nuget package. 

11. Create Nuget Packages
Create a Python script agent with the following settings:
Script path:
$(Build.SourcesDirectory)\scripts\run.py
Argument to  
NugetUwpCI --buildOutputPath $(Build.BinariesDirectory)\Build --nugetOutputPath $(Build.BinariesDirectory)\Nuget --gnOutputPath $(Build.BinariesDirectory)\GN --manualNugetVersionNumber "1.71.0.$(Build.BuildId)"
Working directory to $(Build.SourcesDirectory)
Custom condition: 
eq(variables['config.packNuget'], 'true')
12. Create a WebRtc Feed
Follow the bellow link to create a feed in the Artifact and connect it to the nuget package
https://docs.microsoft.com/en-us/azure/devops/artifacts/get-started-nuget?view=azure-devops&tabs=new-nav#create-a-feed
13. Create Nuget Push
Create a Nuget agent to push the nuget package.
Set the command to push 
Publish the package to 
$(Build.BinariesDirectory)/Nuget/package/*.nupkg;
Target feed to the name of the artifact that is created.
Custom condition:
eq(variables['config.packNuget'], 'true')

