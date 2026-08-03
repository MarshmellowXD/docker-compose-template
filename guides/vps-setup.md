**Free Oracle Cloud VPS**

Introduction:

Oracle Cloud offers a Compute Instance with *2 OCPUs*, *12GB RAM*, and *200GB* storage within their "Always Free" resources. This guide will show you how to set it up. 
To use the Oracle Cloud Free Tier, you will need a valid credit/debit card. Virtual/Prepaid cards are not accepted.

**DANGER**

*Although it's impossible to get charged while on the Free tier, if you upgrade to a Pay-As-You-Go (PAYG) account, you will be charged for any resources you use that are not covered by the Free Tier. Therefore, it is imperative to ensure that you set your instance up exactly as described in this guide, and that you do not use any resources that are not covered by the Free Tier.

*Make sure you do not have any extra instances, volumes etc. created at any given time. The free tier resources cover 200GB of total boot volume space, and exactly 1 A1 Flex instance with 2 OCPUs and 12GB RAM.

*If you create any additional resources, you will be charged for them.

*If you do set everything up correctly, the only thing you can be charged for is bandwidth. Ingress bandwidth is unlimited and free, but egress bandwidth is limited to 10TB per month. If you exceed this limit, you will be charged.


## 1. Account Creation 

Log in and create your account [Oracle Cloud Console](https://cloud.oracle.com)

**Note**  

   Many people tend to have issues with the verification process when signing up for Oracle Cloud. If you face any issues, try one of the following:

*Ensure your billing address matches the address exactly as it appears on your bank statement or banking app.

*Use an inPrivate window on Microsoft Edge
*Use a different email address.

*Use a different card.

*Disable any adblockers/VPNs.

*If you still face issues, you can try using a contacting Oracle support through the live chat widget in the bottom right corner of the Oracle Cloud website.

## 2. Network Configuration (VCN)

Before provisioning a server, you should establish a virtual network environment.

1. Navigate to the Oracle Cloud Console menu ☰, and select **Networking** → **Virtual Cloud** → **Overview**.
2. **Create VCN with Internet Connectivity**→ Click **Start VCN Wizard**  
3. Assign a name to your network (e.g.,`oracle-vps`), keep the default settings, and click **Next** followed by **Create**.


## 3. Setup

**Creating our VM instance**

1. Navigate to **Compute** → **Instances** → **Create instance**.
2. **Name:** Assign an identifiable name to your server.
3. **Image:** Click **Change image** and select **Canonical Ubuntu 24.04** (Standard build).
4. **Shape:** Click **Change shape** and select the **Ampere (ARM)** series. Allocate **2 OCPUs and 12GB Memory** to fully utilize the revised free allowance within a single instance.
5. **Networking:** Ensure the VCN created in the previous step is selected.
6. **SSH Keys:** Download **both the private and public keys**. Securely store these on your local machine, as they are mandatory for initial access.
7. **Boot Volume:** Toggle "Specify a custom boot volume size", allocate **200GB** to maximize available storage, and set the boot volume performance option below to **120 VPU**.
8. Click **Create**.


## ***Troubleshooting "Out of Capacity" Errors**

You may encounter an Out of capacity error when creating your VM instance. This is because the free tier has a limited number of instances available.
You can try again later, or you can also switch to a PAYG account, which will allow you to create instances without any restrictions. As long as you stay within the free tier limits, you won't be charged. Setting up a $1 budget alert to ensure you get notified if you go over the free tier limits is recommended.

**How to upgrade to a PAYG account**:

*Go to the Oracle Cloud dashboard.
*Click on the hamburger menu in the top left corner and scroll down to Billing and Cost Management.
*Under the Billing section, click on Upgrade and Manage Payment
*Click on Upgrade to Pay-As-You-Go and follow the instructions.

**Note**

You will see a $100 charge on your card when you upgrade to a PAYG account. This is a temporary hold and will be released either immediately or within a few days. This is to ensure that you have a valid payment method on file. You will not be charged for anything unless you go over the free tier limits. 
It can take up to 24 hours for the upgrade to take effect. You will receive an email once the upgrade is complete.

**How to set up a $1 budget alert**:

*Go back to the Oracle Cloud dashboard.
*Click on the hamburger menu in the top left corner and scroll down to Billing and Cost Management.
*Under the Cost Management section, click on Budgets.
*Click on Create Budget.
*Use the following settings:
1.Budget Name: dont_charge_me
2.Budget Amount: 1
3.Day of the month to begin budget processing: 1
4.Threshold Metric: Forecast Spend
5.Threshhold Type: Percentage of Budget
6.Threshold: 1%
7.Email Recipients: Your email address
8.Email message: Your current usage exceeds Always Free resources. Please check your usage to avoid charges.
9.Click on Create.

**How to check if my current usage is forecast to exceed the free tier limits**:

Go back to the Oracle Cloud dashboard.
Click on the hamburger menu in the top left corner and scroll down to Billing and Cost Management.
Under the Cost Management section, click on Cost Analysis.
Check the Show Forecast box.
Set the End Date to a future date, such as after a few months.
Click on Apply.
You will now see a forecast of your usage. There will be a graph showing your usage over time, and a table showing your usage by service.
Ensure it is all 0.



## 4. Establishing Server Access

To connect via a standard terminal, you must first change permissions of the private key file. Open a terminal on your local machine:

**For Linux (Kubuntu) / Mac:**
1. Change key permissions:
   `chmod 400 /path/to/your-private-key.key`
2. Initialize the SSH connection using the public IP provided in your Oracle instance details:
   `ssh -i /path/to/your-private-key.key ubuntu@YOUR-ORACLE-IP`

**For Windows (PowerShell):**
1. Change key permissions:
   `icacls.exe "C:\path\to\your-private-key.key" /inheritance:r /grant:r "$($env:USERNAME):(R)"`
2. Initialize the SSH connection using the public IP provided in your Oracle instance details:
   `ssh -i "C:\path\to\your-private-key.key" ubuntu@YOUR-ORACLE-IP`
3. You should see `are you sure you want to continue connecting (yes/no)` type yes and enter, then you should see `ubuntu@yourinstance` 

## 5. Opening Ports:

**Note**

Oracle maintains a secondary, network-level firewall. You must mirror your port allowances here, otherwise, inbound traffic will be blocked prior to reaching the OS. It is recommended to configure this before enabling your local OS firewall to prevent locking yourself out. 

**minimize your terminal, we will come back to it shortly.**

1. In the Oracle Cloud Console, navigate to your Instance → **Networking** → click your VCN name.
2. Select the public subnet, then click the **Default Security List**.
3. Click **Add Ingress Rules**.
4. Configure the Source CIDR as `0.0.0.0/0` and create rules for the following Destination Port Ranges:
   * `80` (HTTP)
   * `443` (HTTPS)

## 6. Setup Steps:

Now back to your terminal! We will now update the operating system and configure the local Firewall (UFW) to permit essential traffic.

```bash
# First lets update system packages
sudo apt update && sudo apt upgrade -y

# Now Install UFW
sudo apt install ufw -y

# Allow the essential ports
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Enable the firewall
sudo ufw enable
```

This is all for setting up your vps. To deploy the template see [Template Deployment Guide](https://github.com/MarshmellowXD/docker-compose-template/blob/main/guides/template-deployment.md)








