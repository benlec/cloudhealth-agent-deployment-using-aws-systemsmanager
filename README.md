This article explain how you can easily deploy your CloudHealth agent using System Manager and ensure compliance of your installation using AWS State Manager.

At the end of the article you will have achieved the following goals:
* Deploy CloudHealth agent Package on all platforms (Windows, Amazon Linux, Ubuntu)
* Control and manage the correct state of your instances making sure CloudHealth agent is deployed everywhere on any instances even the new ones

What we will do in order
* Create the CloudHealth agent Package for all platforms including Install and Uninstall Scripts
* Upload the package in AWS Systems Manager Distributor
* Create an Association with State Manager (System Manager) to constantly control that CloudHealth agent is present and automatically install it if it’s not the case

## 1. Create your CloudHealth agent Package for Distributor (System Manager)

AWS Systems Manager Distributor lets you package your own software to install on AWS Systems Manager managed instances.
As per as the documentation you first need to create a Zip file containing your install and uninstall scripts as well as the files you want to install on your instances in order to use it in Distributor
Please have a look at the installation instructions on the CloudHealth documentation before proceeding

**1.1 Create the Windows Package:**

Windows package will contain three files:
* Install.ps1
* Uninstall.ps1
* CloudHealthAgent.exe

Put the following text in Install.ps1 file:

```PowerShell
Write-Output 'Installing CloudHealthAgent on Windows...'
Start-Process CloudHealthAgent.exe -ArgumentList "/s, /v`"/l* install.log /qn CLOUDNAME=aws CHTAPIKEY=YOUAPIKEYHERE`""
```

Put the following text in Uninstall.ps1 file:

```PowerShell
Write-Output 'Uninstalling CloudHealthAgent on Windows...'
Start-Process CloudHealthAgent.exe -ArgumentList "/x /s /v/qn" 
```

Get the CloudHealthAgent.exe files as explained in the documentation

Create a zip file with these 3 files, you will end-up with the following zip file content:
```
CloudHealthPackage_WINDOWS.zip
			|-------- install.ps1
			|-------- uninstall.ps1
			|-------- cloudhealthagent.exe
```

**1.2 Create the Ubuntu and Amazon Linux Packages:**

Amazon Linux and Ubuntu packages will each contain only two files:
* Install.sh
* Uninstall.sh

Both files can be the same for Amazon Linux and Ubuntu

Put the following text in install.sh file:
```bash
#!/bin/bash
echo 'Installing CloudHealthAgent on Amazon Linux...'
wget https://s3.amazonaws.com/remote-collector/agent/v20/install_cht_perfmon.sh -O install_cht_perfmon.sh;
sudo sh install_cht_perfmon.sh 20 YOUR_KEY_HERE_AS_PER_AS_CH_DOCUMENTATION aws;
```

Put the following text in uninstall.sh file:
```bash
#!/bin/bash
echo 'Uninstalling CloudHealthAgent on Amazon Linux...'
wget -O - https://s3.amazonaws.com/remote-collector/agent/uninstall_cht_perfmon.sh | sudo sh
```

Create two zip files with these 2 files you will end-up with the following two zip files content:

```
CloudHealthPackage_AMAZON.zip
                    |-------- install.sh
                    |-------- uninstall.sh
CloudHealthPackage_UBUNTU.zip
                    |-------- install.sh
                    |-------- uninstall.sh
```
**1.3. Create your package Manifest and finalize your AWS System Manager Distributor package**

Create a file called ___manifest.json___, add it to the folder where you created the other zip files. In the end you should have a folder containing the following files:

```
MyCloudHealthPackageFolder
		    |-------- CloudHealthPackage_AMAZON.zip
		    |-------- CloudHealthPackage_ AMAZON.zip
		    |-------- CloudHealthPackage_UBUNTU.zip
		    |-------- manifest.json
```

For your ___manifest.json___ file you can simply use the same content as the one provided in the AWS documentation. At the beginning of the documentation there is a link to download the ExamplePackage.zip file.
You just need to modify the following in the provided JSON file
* The name of the Zip files (6 references in total)
* The checksums for each Zip file (3 references in total)

Also be careful to note the version of the file in the manifest as you will need to use the same in AWS Systems Manager Distributor
```json
Ex:     "version": "1.0",
```
## 2. Upload your Package in S3 and then in AWS System Manager Distributor

**2.1.	Simply create a bucket and upload all your files to the S3 bucket, you should end-up with something like this**
 

**2.2.	Now create you package in Distributor**

* Open AWS System Manager Console, go to Distributor on the left Menu (Under the “Actions” section)
* Click on “Create Package”, give your package a name like “CloudHealthAgentPackage”
* Set the version to the same number you set in your Manifest.json as explained at the end of section 1.3 of this document
* Specify the URL of the bucket you created in Step 2.1 of this document
* For the manifest, select “Extract from package” and click “View Manifest File”, manifest file content will appear then
* Click “Create Package” and wait for your package to be created successfully on next screen

## 3.Create an Associate using State Manager to install this package and ensure compliance of this installation

**3.1.	Create your Association**

* Open AWS System Manager Console, go to ___“State Manager”___ on the left Menu (Under the “Actions” section)
* Click on ___“Create Associate”___, give it a name like ___“CloudWatchAgentInstallationState”___
* Under ___“Document”___ Select ___“AWS-ConfigureAWSPackage”___ document with the radio button
* Set the following ___“Parameters”___
  * ___Action:___ Install
  * ___Name:___ CloudHealthAgentPackage (the name shall match exactly the name you used in Distributor)
  * ___Version:___ 1.0 (shall exactly match the version you used in Distributor)
* Under ___“Targets”___ select ___“All managed instances”___ if you want all your EC2 to get your package (recommended)
* Under ___“Specify Schedule”___, set let the default schedule of 30 minutes or change it as you please
* Choose a compliance severity and click ___“Create Association”___

**3.2 Wait for the association to execute once**

Verify that everything is fine under the compliance section.

State Manager allow you to do two main things in this scenario

* Verify that your package is installed as required
* Install package if the association is not compliant (package not installed)

___Using System Manager here allow you to get the full benefit of CloudHealth recommendations in terms of Right Sizing as it is required to deploy the Agent on all instances in order to collect all needed data to have precise insights of the EC2 instances utilization and make the right decisions.___








