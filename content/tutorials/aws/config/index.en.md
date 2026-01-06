---
title: "Configuring your AWS environment"
tags: ["AWS"]
date: 2026-01-05
draft: false
---

Once you've created your AWS account, it's considered a best practice not to use your main account for day-to-day tasks. To avoid using the main account, it's recommended to create an alternative user with access to the necessary resources

In our case, we can list specific resources, but since the goal is to study AWS resources, I created a new user with administrative resources

Once logged into the main account, search for Identity Access Management (IAM) to create the new user
![IAM](img/user1.png)

## Create a new user
* Give your user a name and check the option <i><strong>Provide user access to the AWS Management Console - optional</strong></i> so the user can log into the AWS console
* In my specific case, I want to create a user with which I can deploy resources from my personal machine with IaC (Infrastructure as Code), so I checked the <i><strong>AdministratorAccess</strong></i> policy
* Create the user by selecting the <i><strong>Create User</strong></i> button
* After creating the user, you can retrieve the temporary password

{{<gallery>}}
  <img src="img/user2.png" class="grid-w25">
  <img src="img/user3.png" class="grid-w25">
  <img src="img/user4.png" class="grid-w25">
  <img src="img/user5.png" class="grid-w25">
{{</gallery>}}

## Get the Access Key
Once your user has been created, select the user and the <b><i>Access Key</i></b> field on the screen
{{<figure src="img/access-key1.png">}}

Select the <b><i>Command Line Interface (CLI)</i></b> option and download the csv file or copy the access key. This information will be necessary for configuring AWS credentials in your development environment
{{<gallery>}}
<img src="img/access-key2.png" class="grid-w25">
<img src="img/access-key3.png" class="grid-w33">
<img src="img/access-key4.png" class="grid-w33">
{{</gallery>}}

<hr />

## First Login

> [!info]+ First Login
> Once the user has been created, make the first access and generate the new password

{{<gallery>}}
  <img src="img/login1.png" class="grid-w33">
  <img src="img/login2.png" class="grid-w33">
{{</gallery>}}

<hr />

## Configure the local environment

Once you've created the new user and obtained the access key, we can then create the necessary files for the local environment

{{<tabs>}}
  {{<tab label="Linux / macOS">}}
  ```shell
  mkdir ~/.aws
  cd ~/.aws
  
  echo [default] > credentials
  echo aws_access_key_id=<access key> >> credentials
  echo aws_secret_access_key=<access secret> >> credentials

  echo [default] > config
  echo region=us-east-1 >> config
  echo output=json >> config
  ```
  {{</tab>}}
  {{<tab label="Windows">}}
  ```powershell
  mkdir C:\Users\USERNAME\.aws
  cd C:\Users\USERNAME\.aws
  
  echo [default] > credentials
  echo aws_access_key_id=<access key> >> credentials
  echo aws_secret_access_key=<access secret> >> credentials

  echo [default] > config
  echo region=us-east-1 >> config
  echo output=json >> config
  ```  
  {{</tab>}}
{{</tabs>}}