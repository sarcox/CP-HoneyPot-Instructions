# Weeks 10 & 11 Project: Honeypot

**Summary**: Setup a honeypot and intercept some attempted attacks in the wild.
<img src="https://i.imgur.com/QEzVHda.png">

## Background
A honeypot is a decoy application, server, or other networked resource that intentionally exposes insecure features which, when exploited by an attacker, will reveal information about the methods, tools, and possibly even the identity of that attacker. Honeypots are commonly used by security researchers to understand the threat landscape facing developers and system administrators, collecting data that might include:

+ Information about sources of malicious network traffic such as IP addresses, geographic origin, targeted ports, etc.
+ Information used to harden resources against email spammers
+ Malware samples
+ DB vulnerabilities such as SQLI techniques

There are two broad categories of honeypots:
+ **Low-interaction honeypots** provide simulations of target resources, typically using emulation or virtualization, to reduce resource consumption, simplify configuration and deployment, and provide solid containment features
+ **High-interaction honeypots** expose non-simulated target resources in a way that more closely imitates a production environment to attract more sophisticated attackers and understand more complicated exploitation routes

For example, a low-interaction honeypot might emulate a server capable of accepting SSH connections through a combination of exposed ports and decoy responses, whereas the high-interaction version would feature an actual SSH server possibly misconfigured in some way that makes it vulnerable. In the low-interaction example, attempted exploitation would quickly lead to a dead end for the attacker, perhaps revealing only an IP address and a few attempted commands to the honeypot's maintainer. In the high-interaction example, the attacker would potentially be able to compromise the server, wasting more time and giving away more information about his or her goals.

## Overview
In this assignment, you will stand up a basic honeypot and demonstrate its effectiveness at detecting and/or collecting data about an attack. Guided instructions for doing this using specific software are provided below, but you are free to take any approach you wish that demonstrates the following basic principles:

Suc+ cessful configuration and deployment of a network-accessible honeypot server with two primary features:
++ An attack surface that is vulnerable or exposed in some way to network-based attacks
+ A network security feature such as an IDS configured to detect and log such attacks
Illustration of at least one attack against the honeypot that can be detected or logged in a way that captures information about the attack or the attacker
<hr>

# Walkthrough

<hr>
Keeping in mind that there are many ways one could fulfill the above requirements, in this section, we will walkthrough a basic honeypot deployment using a well-supported open source honeypot: <a href="https://github.com/pwnlandia/mhn">Modern Honey Network (MHN)</a>. MHN's architecture is modular and extensible and comes with many options for deploying different types of honey pots. In MHN architecture, there is a single admin VM which is used to deploy, manage and collect information from the honeypots, which are deployed as separate VMs. Thus to run MHN, we'll need to setup at least two VMs: the single **Admin VM** and at least one **Honeypot VM**.

** Milestone 0: To the Google Cloud!
To complete this assignment, you'll need access to a cloud hosting provider to provision the VMs. Many providers offer time-limited free trial accounts with resource limitations, and you should easily be able to complete the requirements for this assignment within these limitations -- though you may need to ensure you cleanup before your trial period expires. The setup we'll walkthrough below has been tested to work with standard sized VMs. Running MHN Admin on micro size VM's is not recommended.

You can use any cloud provider to which you already have access or that offers a free trial, though you'll need to be familiar with its usage and / or limitations. If you're not sure where to start, we recommend <a href="https://cloud.google.com/free/">Google Cloud Platform's Free Tier</a>, and while we'll provide general guidelines that should work with most cloud providers, the instructions below will also show insets labeled **GCP Users** with commands and settings specific to Google Cloud Platform. If you are confident about working with an alternate cloud provider such as AWS feel free to adapt the below instructions accordingly. If this is your first foray into the world of cloud computing, consider starting a GCP trial so you can follow the more specific instructions below.

## Setting up your GCP environment
You will complete some of the initial setup steps in the Web UI for GCP. This should give you a feel for the tools and functionality GCP provides.

1. Navigate to https://console.cloud.google.com/. Log on if necessary.
    If you have multiple Google Accounts, make sure you are signed in with the account you would like ot use for this project.
2. Click the **TRY FOR FREE** button.
  Fill out the information including providing a credit card number. The VMs you create for this project will not use up even 25% of your free credits if you create them as described here and only run them for a week or 2. Google will not autocharge you if you go over your limit.
  
  Once you are enrolled you will land on the Welcome page.
  
3. Create a new project for this assignment.
    1. Click the **My First Project** menu to bring up a list of your projects.
    1. Click the **NEW PROJECT** button.
    1. Give the project a name and click **CREATE**.
    You will see a notification that the project is being created. This may take a minute.
    
4. Click the **GO TO COMPUTE ENGINE** link. You will use this page to create your VMs. It may take a few minutes to prepare your Compute Engine.

## Milestone 1: Create MHN Admin VM
**Summary:** Start by creating the MHN Admin VM via your cloud provider. The VM needs to have an internet-facing IP and accessible to you via SSH. The MHN Admin VM will be provisioned with the following attributes:

+ Ubuntu 18.04 Minimal
+ HTTP traffic allowed (port 80)
+ TCP ports 3000 and 10000 need to be open to allow incoming (aka 'ingress') traffic for geolocation and honeypot sensor data.

That last requirement is generally the only one that may require a specific firewall rule to configure properly, because those ports are non-standard and specific to MHN. Some cloud providers may require you to create the firewall rules separately and then apply them to the VM. Either way, make sure when you create the VM that you can access it via SSH.

1. Click the **Create** button to create a new VM. You will create a VM to administrate the honeypot project.
  1. Name the VM *mhn-admin*. This specific name is important because you will copy and paste commands later in the setup that refer to this VM name.
  1. Set the **Region** to us-west 1. The Zone will automatically update to a related zone, such as us-west1-b.
  1. Change the **Machine type** to f1-micro. Creating the correctly sized system means you do not waste your compute credits on systems that are more powerufl than you need.
  1. In the **Boot disk** seciton, click the **Change** button. 
  1. Choose the **Ubuntu 14.04 LTS** image, then click **Select**.
  1. In the **Firewall** section, select the boxes for **Allow HTTP traffic** and **Allow HTTPS* traffic.
  1. Click **Create**.
    
2. Configure your firewall rules. You can do this in the Web UI, but it's very fast to do through the console.
  1. Click the **Access Cloud Shell** button in the header bar of GCP. 
  <img src="https://github.com/sarcox/CP-HoneyPot-Instructions/blob/master/GCP-CloudShell.png">
Alternatively, you can <a href="https://cloud.google.com/sdk/docs/quickstarts">download and install the GCP SDK</a> so you can SSH directly to youur environment.
  1. Run the following commands in the SDK to create the appropriate firewall rules. Note the target-tags point at the mh-admin VM you created above.

```
    gcloud compute firewall-rules create honeymap \
    --allow tcp:3000 \
    --description="Allow HoneyMap Feature from Anywhere" \
    --direction ingress \
    --target-tags="mhn-admin"

    gcloud compute firewall-rules create hpfeeds \
    --allow tcp:10000 \
    --description="Allow HPFeeds from Anywhere" \
    --direction ingress \
    --target-tags="mhn-admin"
 ```
   1. (Optional) Review the firewall rules you created.
     + In the browser, you 
