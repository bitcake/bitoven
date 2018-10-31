# BitOven

BitOven is [BitCake](http://bitcakestudio.com/)'s custom Continuous Integration (CI) solution.

## List of Requirements
- An unlocked [AWS](https://aws.amazon.com/) account (Not in free trial mode, we use big machines)
- IAM Access credentials (Key + Secret Key pair) with the following permissions:
  - Administrative Access (to be changed to less intrusive permissions in the future)
- Access credentials for Unity login (username + password), with a unity plus or higher seat available, with the serial number
- For Steam uploading, a steam account added to the company partner website with the following permissions on the account:
  - Edit App Metadata
  - Publish App Changes to Steam

## Overview

BitOven is based on a serverless structure on AWS that supports varying workloads and configurations for your Unity Projects.

It consists of 5 main components:

1. The commit hook
    - A serverless function that handles your repository webhooks (currently supports bitbucket with mercurial)
2. A processing queue
    - That handles messages about important commits and orchestrates all the build requests
3. A DynamoDB Table with all the Build Targets your studio wants built
    - So for each of your repositories you can specify any number of build targets you want, each of them has their own sets of commands to execute, can use a specific version of an AMI or a different sized machine. Want a ultra fast urgent build, fire up a xtra large machine instead of a regular one. 😉
4. A collection of Amazon AMIs for actually building stuff
    - Machine templates that are configured to load up your projects and do the dirty work
5. A build plugin (*.unitypackage) for your Unity Projects
    - The plugin is highly extensible and exposes some endpoints for your project to be built by the command line

## The AMIs

AMIs are pre-built machine images with the core of BitOven already setup into it. 
Each AMI has it’s own Unity version and BitOven core version. You should use the correct version for your project in each build target.

### Example:
> Your studio has 2 different projects for CI, one of them uses Unity 2018.1, the other one uses Unity 2017.4.1f1. So for each  project you would look at the list of available AMIs in this document and find the id of the corresponding AMI in order to use it in the appropriate field.

New AMIs with specific versions can be requested at jefferson@bitcakestudio.com we’ll do our best to accommodate reasonable requests.

### Available AMIs
- ami-084b2070: Version 1.0.2 Unity 2017.4.1f1 (bitbucket, xp-dev mercurial keys)

## The AWS Stack

BitOven has some key components on the AWS system that are worth noting:

### S3 Bucket - keybucket

This S3 bucket should store ssh keys and steam guard auth files that will be read by the build workers. Visibility is private to the AWS account

### S3 Bucket - logbucket

This S3 bucket will hold all build logs generated, and will also hold the actual builds if UploadToS3 is set on a Recipe. Default access for this bucket is configured as public, but can be changed to private if needed.

### DynamoDB Table - Build Recipes

This table will hold all Recipe configurations

## Build Recipes

BitOven orchestrates the builds based on data called BuildRecipe, each AWS Stack has a DynamoDB table inside it where you can configure each Recipe. A Recipe consists of:

### Data Structure
- **Enabled**: A bool that enables/disables the recipe being checked for builds or not.
- **Repository Name**: Name of the repo that this target refers to (must be the same as the bitbucket repo)
- **Type**: “Message”, ”Tag” or “Branch”. Currently only message works, but “Tag” and “Branch” are being worked on
- **Target**: The message/tag/branch that will trigger the build
    Example: If RepositoryName is “Holodrive”, Type is “Message” and Target is “[steam]”, all pushed commits that have ‘[steam]’ in the commit message will trigger a new build.
- **NeededSize**: A number, in gigabytes, for the project storage container. This should be big enough to fit your project (with the library folder) + a built version of the project.
- **SlackWebhookUrl**: the url that slacks generates that enables BitOven to notify your team when the build is ready.
- **UploadToS3**: true/false if the build output should automatically be uploaded to the S3 logbucket
- **RepositoryInfo**:
  - **Type**: “Mercurial” or “Git”. 
      Currently only “Mercurial” is implemented, git support still pending.
  - **Url**: URL of the bitbucket repository for ssh access. 
      (usually ssh://hg@bitbucket.org/company/repo).
  - **KeyName**: Name of a PuTTY key that has been uploaded to the S3 keybucket that allows ssh access to the repository for cloning/pulling changes.
- **BuildData**:
  - **ImageId**: the AMI ID that should be used for this Recipe
  - **InstanceType**: Type of instance that will be spawned to build this Recipe
      Ex.: c5.large, t2.micro, m5.xlarge
  - **ProjectSubFolder**: If your project root is not at the repository root, specify the path here
  - **BuildCommands**: List of commands to be passed on to Unity, one per line. This will usually include things like username/password/serial for license authentication and which commands will be executed to actually create a build
- **SteamUploadData** - Include this if your project will do automatic steam uploads
  - **SteamAuthFilenames**: List of auth files that have been uploaded to the S3 keybucket for login with the steam cmd without requiring steam guard.

You can find a skeleton example here: https://github.com/bitcake/bitoven/blob/master/build-recipe-example.json

## Steam upload auth files

For the build machines to be able to upload your builds to steam, you have to provide auth files that will let it bypass the steam guard procedure. You can get these following those steps:


- Log in to SteamCMD as if you’re doing a normal build
- Enter your credentials and steam guard auth code
- Exit SteamCMD
- Look for 2 files that starts with ”ssfn” in the SteamCMD directly (same folder as the executable)
- Upload both files to the keybucket and save their names
- Add the name of both files to the SteamAuthFilenames array on your build recipes

