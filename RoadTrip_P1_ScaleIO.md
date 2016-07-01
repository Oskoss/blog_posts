# Road trip to Persistence on CloudFoundry

Over the past few months the Dojo has been working with all the types of storage to enable persistence within CloudFoundry. Across the next few weeks we are going to be road tripping through how we enabled EMC storage on the CloudFoundry platform. For our first leg of the journey, we start with ScaleIO, a software defined storage service that is both flexible to allow dynamic scaling of storage nodes as well as reliable to enable enterprise level confidence.

## Chapter 1 - ScaleIO

### What is ScaleIO -- SDS, SDC, & MDM!?

ScaleIO as we already pointed out is a software defined block storage. In laymen terms there are two huge benefits I see with using ScaleIO. Firstly, the actual storage backing ScaleIO can be dynamically scaled up and down by adding and removing SDS (ScaleIO Data Storage) server/nodes. Secondly, SDS nodes can run parallel to your applications running on a server, utilizing any additional free storage your applications are not using. These two points allow for a fully automated datacenter and a terrific base to start for block storage in CloudFoundry.

Throughout this article we will use SDS, SDC, and MDM, lets define them for some deeper understanding!
All three of these terms are actually services running on a node. These nodes can either be a Hypervisor (in the case of vSphere), a VM, or a bare metal machine.

##### SDS - ScaleIO Data Storage

This is the base of ScaleIO. SDS nodes store information locally on storage devices specified by the admin.

##### SDC - ScaleIO Data Client

If you intend to use a ScaleIO volume, you are required to become an SDC. To become an SDC you are required to install a kernel module (.KO) which is compiled specially for your specific Operating system version. These all can be found on [EMC Support](http://support.emc.com). In addition to the KO that gets installed there also will be a handy binary, drv_cfg. We will use this later on but make sure you have it!

##### MDM - Meta Data Manager

Think of the MDMs as the mothers of your ScaleIO deployment. They are the most important part of your ScaleIO deployment, they allow access to the storage (by means of mapping volumes from SDS's to SDC's), and most importantly they keep track of where all the data is living. Without the MDM's you lose access to your data since "Mom" isn't there to piece together the blocks you have written! Side Note: make sure you have at least 3 MDM nodes. This is the smallest number allowed since it is required to have 1 MDM each for Master, Slave, and Tiebreaker.


### How to Install ScaleIO

The number of different ways to install ScaleIO is unlimited! In the Dojo we used two separate ways, each with their ups and downs. The first, "The MVP", is simple and fast, and it will get you the quickest minimal viable product. The second method, "For the Grownups", will provide you with a start for a fully production ready environment. Both of these will suffice for the rest of our road tripping blog.

##### The MVP

This process uses a Vagrant box to deploy a ScaleIO cluster. Using the [EMC {Code}](http://emccode.com/) ScaleIO vagrant [Github Repository](http://github.com/emccode/vagrant/tree/master/scaleio), checkout the ReadMe to install ScaleIO in less than an hour (depending on your internet of course :smirk: ). Make sure to read through the `Clusterinstall function` of the ReadMe to understand the two different ways of installing the ScaleIO cluster.

##### For the GrownUps

This process will deploy ScaleIO on four separate Ubuntu machines/VMs.

*Checkout The [ScaleIO 2.0 Deployment Guide](http://support.emc.com/docu67398_ScaleIO-2.0-Deployment-on-Windows-and-Linux-Quick-Start-Guide.pdf?language=en_US) for more information and help*

* Go to [EMC Support](http://support.emc.com).
  *  Search ScaleIO 2.0
  *  Download the correct ScaleIO 2.0 software package for your OS/architecture type.
    * **Ubuntu** (We only support Ubuntu currently in CloudFoundry)
    * RHEL 6/7
    * SLES 11 SP3/12
    * OpenStack
  * Download the ScaleIO Linux Gateway.
* Extract the \*.zip files downloaded

**Prepare Machines For Deploying ScaleIO**

* Minimal Requirements:
  * At least 3 machines for starting a cluster.
    * 3 MDM's
    * Any number of SDC's
  * Can use either a virtual or physical machine
  * OS must be installed and configured for use to install cluster including the following:
    * SSH must be installed, and be [available for root](http://askubuntu.com/questions/469143/how-to-enable-ssh-root-access-on-ubuntu-14-04). Double-check that passwords are properly provided to configuration.
    * libaio1 package should be installed as well. On Ubuntu: `apt-get install libaio1`


**Prepare the IM (Installation Manager)**

  * On the local machine SCP the Gateway Zip file to the Ubuntu Machine.
  ```
  scp ${GATEWAY_ZIP_FILE} ${UBUNTU_USER}@${UBUNTU_MACHINE}:${UBUNTU_PATH}
  ```


 * SSH into Machine that you intend to install the Gateway and Installation Manager on.
  * Install Java 8.0
    ```
    sudo apt-get install python-software-properties
    sudo add-apt-repository ppa:webupd8team/java
    sudo apt-get update
    sudo apt-get install oracle-java8-installer
    ```
  * Install Unzip and Unzip file
    ```
    sudo apt-get install unzip
    unzip ${UBUNTU_PATH}/${GATEWAY_ZIP_FILE}
    ```
  * Run the Installer on the unzipped debian package
    ```
    sudo GATEWAY_ADMIN_PASSWORD=<new_GW_admin_password> dpkg -i ${GATEWAY_FILE}.deb
    ```
  * Access the gateway installer GUI on a web browser using the Gateway Machine's IP. `http://{$GATEWAY_IP}`
  * Login using `admin` and the password you used to run the debian package earlier.
  * Read over the install process on the Home page and click `Get Started`
  * Click browse and select the following packages to upload from your local machine. Then click `Proceed to install`
    * XCache
    * SDS
    * SDC
    * LIA
    * MDM

    Installing ScaleIO is done through a CSV. For our demo environment we run the minimal ScaleIO install. We built the following install CSV from the minimal template you will see on the Install page. You might need to build your own version to suit for your needs.
    ```
    IPs,Password,Operating System,Is MDM/TB,Is SDS,SDS Device List,Is SDC
    10.100.3.1,PASSWORD,linux,Master,Yes,/dev/sdb,No
    10.100.3.2,PASSWORD,linux,Slave,Yes,/dev/sdb,No
    10.100.3.3,PASSWORD,linux,TB,Yes,/dev/sdb,No
    ```
  * To manage the ScaleIO cluster you utilize the MDM, make sure that you set a password for the MDM and LIA services on the Credentials Configuration page.
  * NOTE: For our installation, we had no need to change advanced installation options or configure log server. Use these options at your own risk!

* After submitting the installation form, a monitoring tab should become available to monitor the installation progress.

  * Once the Query Phase finishes successfully, select start upload phase. This phase uploads all the correct resources needed to the nodes indicated in the CSVs.
  * Once the Upload Phase finishes successfully, select start install phase.
  * Installation phase is hopefully self-explanatory.  

* Once all steps have completed, the ScaleIO Cluster is now deployed.
  * To start using the cluster with the ScaleIO cli you can follow the below steps which are copied from the post installation instructions.

    To start using your storage:
    Log in to the MDM:

    `scli --login --username admin --password <password>`

    Add SDS devices: (unless they were already added using a CSV file containing devices)
    You must add at least one device to at least three SDSs, with a minimum of 100 GB free storage capacity per device.

    `scli --add_sds_device --sds_ip <IP> --protection_domain_name default --storage_pool_name default --device_path /dev/sdX or D,E,...`

    Add a volume:

    `scli --add_volume --protection_domain_name default --storage_pool_name default --size_gb <SIZE> --volume_name <NAME>`

    Map a volume:

    `scli --map_volume_to_sdc --volume_name <NAME> --sdc_ip <IP>`
**Managing ScaleIO**

When using ScaleIO with CloudFoundry we will use the ScaleIO REST Gateway to manage the cluster. There are other ways to manage the cluster such as the ScaleIO Cli and ScaleIO GUI, both of which are much harder for CloudFoundry to communicate with.

### EOF

At this point you have a fully functional ScaleIO cluster that we can use with CloudFoundry and RexRay to deploy applications backed by ScaleIO storage! Stay tuned for our next blog post in which we will deploy a minimal CloudFoundry instance.

Cloud Foundry. Open Source. The Way. EMC [⛩] Dojo.
