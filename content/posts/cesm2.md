+++
title = 'How to install and run CESM2 in the cloud with AWS'
date = 2024-01-05T17:44:22-08:00
draft = false
+++

Supercomputers around the world are dedicated to the computational prediction and analysis of climate scenarios. These supercomputers run state-of-the-art models such as NCAR’s [Community Earth System Model](https://www.cesm.ucar.edu/). While, CESM2 is available to researchers around the world, the compute time required to run it is not always easy to find. Cloud computing offers an alternative. 
This blog describes how to install and run CESM2 on AWS using an AWS service called AWS Parallelcluster. 

## What is AWS ParallelCluster? 
AWS ParallelCluster is an open source cluster management tool installed and run by a system administrator to create a compute cluster in an AWS account. CESM2 is installed on the cluster and made available to CESM2 users. The cluster environment setup includes a headnode, a SLURM queuing system, storage systems, and "on demand" compute nodes. Often the system administrator is simply the user in need of extra computational resources. :wink:

It is important to distinguish that AWS ParallelCluster is not the cluster but a tool that creates one or many clusters in the designated AWS account based on a yaml configuration file. The yaml configuration file defines the compute nodes to be used, queue names, headnode size, and shared storage for the cluster. AWS ParallelCluster can be installed to a laptop outside of AWS, in an AWS account using AWS cloudshell, AWS Cloud9, or on a separate EC2 instance. AWS ParallelCluster can also be run from the very nice [ParallelCluster UI](https://blogs.silverliningresearch.org/posts/demo/). Unfortunately, as of this writing, the UI does not directly support the version of AWS FSx Lustre used in the recommended yaml file below. The appendix shows a work-around that allows the ParallelCluster UI to be used with this blog.

## Before Getting Started
This blog describes the creation of a cluster, installation of CESM2, followed by brief noes on how to run CESM2. This is not a tutorial for running CESM2 nor does it describe how or where to install AWS ParallelCluster. As a prerequisite to this blog, an AWS account, introductory knowledge of AWS ParallelCluster and an existing installation of AWS ParallelCluster is assumed. The background for that can be found [here](https://www.hpcworkshops.com/08-efa.html). Knowledge of CESM2 is also assumed and while a small demonstration case is included, the details and complexity of running CESM2 requires specific climate science expertise. 

The machine files discussed below allow CESM2 to run on the AWS HPC6a instance type. The HPC6a instance type is designed specifically for HPC calculations and is described [here](https://aws.amazon.com/ec2/instance-types/hpc6/). AWS accounts have service limits to prevent accidental usage (and accidental charges) for features an account owner might not want. As such, new accounts must request an increase to the number of accessible HPC instances.

Double check AWS limits on the AWS Service Quota panel from the AWS console. The hpc6a limit is found as part of the "Ec2,Running On-Demand HPC instances” quota name from the Amazon Elastic Cloud Compute dashboard card. The value quoted are "vcpus", divide by 96 to equal the number of HPC6a instances available. Request a limit increase if necessary before proceeding.

An AWS SSH key is required to launch the cluster and can be created from the Amazon EC2 dashboard if one is not already available. Be sure to secure the private portion of the key as Amazon does not retain this and it can only be downloaded once.

## Cost to Work Through this Blog
The cost to work through this blog is expected to be less than $15. The cost of full-fledged runs can be estimated with the [aws cost calculator](calculator.aws) by timing a short duration version of the desired run. 

## Creating the Cluster 
The steps below walk the user through launching a cluster, installing required libraries, fetching CESM2, and launching a case. 

### Step 1: Create a cluster with the following yaml config file 
The large black box immediately below contains the yaml configuration file needed to launch a cluster to your AWS account from your ParallelCluster installation. Copy and paste the yaml config file to where AWS ParallelCluster is installed. Name the config file "cesm.yaml".

#### Make the following edits:
* Replace the subnet placeholder, subnet-XXXXX, with a subnet where HPC6a is available. Most commonly this will be us-east-2, AZ2. The HPC6a instance is not available in most regions or AZs. The subnet name that corresponds to us-east-2, AZ2 can be found in the VPC section of the AWS console. The subnet needs to be updated in two spots within the configuration file.
* Substitute the name of your ssh key in the place of "addkeynamehere". The ssh key used needs to be in the same region as the subnet. In the yaml configuration file below, this is us-east-2.

#### AWS ParallelCluster YAML Configuration 
```bash
Region: us-east-2
Image:
  Os: alinux2
HeadNode:
  InstanceType: c6a.xlarge
  Networking:
    # change subnet to a subnet in AZ2 in Ohio
    SubnetId: subnet-XXXXX
  Ssh:
    # change to a keyname in your account
    KeyName: addkeynamehere
  DisableSimultaneousMultithreading: true
  LocalStorage:
    RootVolume:
      Size: 75
      VolumeType: gp3
  Iam:
Scheduling:
  Scheduler: slurm
  SlurmQueues:
    - Name: compute
      CapacityType: ONDEMAND
      ComputeResources:
        - Name: compute-hpc6a
          Instances:
            - InstanceType: hpc6a.48xlarge
          MinCount: 0
          # Ensure that the MaxCount is within the account limit for hpc6a
          MaxCount: 11
          DisableSimultaneousMultithreading: true
          Efa:
            Enabled: true
            GdrSupport: true
      Networking:
        SubnetIds:
        #change subnet to a subnet in AZ2 in Ohio
          - subnet-XXXXX
        PlacementGroup:
          Enabled: true
      ComputeSettings:
        LocalStorage:
          RootVolume:
            VolumeType: gp3
            Size: 35
      AllocationStrategy: lowest-price
      Iam:
  SlurmSettings: {}
SharedStorage:
  - MountDir: /opt/ncar
    Name: Ebs0
    StorageType: Ebs
    EbsSettings:
      Size: '25'
      VolumeType: gp3
      DeletionPolicy: Delete
  - MountDir: /scratch
    Name: FsxLustre0
    StorageType: FsxLustre 
    FsxLustreSettings:
      StorageCapacity: 6000
      PerUnitStorageThroughput: 12
      DeploymentType: PERSISTENT_1
      DataCompressionType: LZ4
      StorageType: HDD
      DeletionPolicy: Delete
```

The cluster will fail if the maximum number of instances given by MaxCount exceeds the number of hpc6a.48xlarge instances available. 
Double check that the max number of compute nodes requested in the yaml config file (MaxCount) are within your AWS account limits. 

The yaml config file includes two shared volumes to be attached to the cluster. The first is an Amazon EBS volume (general purpose SSD-based) which is attached at /opt/ncar and allows ephemeral compute node access to CESM2 and the libraries. The second an Amazon FSx for Lustre volume offers /scratch space on a shared 6TB PERSISTENT 1 HDD. The HDD volume is performant enough for CESM2 runs while offering an excellent price point. CESM2 users often require large amounts of scratch space and the Amazon FSx for Lustre volume can be increased from the console in increments of 6TB. 

The /scratch volume is included with a DeletionPolicy equal to "Delete". It is common to change this to Retain if it is desired that the volume not be deleted.

If the ParallelCluster UI is being used, the default is the SSD FSx volume type which is more expensive. Please skip to the Appendix to incorporate the HDD volume type.

To complete this step, launch the modified yaml config file with ParallelCluster and wait for the cluster creation to complete. 

```bash
$ pcluster create-cluster —cluster-configuration cesm.yaml —cluster-name cesm —region us-east-2
```
:dollar: **AWS charges begin at this time and continue until the cluster is deleted as described below.** :money_with_wings:

The new cluster will take approximately 20 minutes to come up. Logging onto the headnode before the cluster has completed its creation can occasionally have unpredictable results.



### Step 2: Install the libraries
	
Congratulations, the cluster should now be created. Log onto the cluster headnode using one of the following methods.


*Log on with ParallelCluster:*
```bash
pcluster ssh --cluster-name cesm -i /path/to/keyfile.pem
```

*Log on with the ParallelCluster UI:*

Even if the ParallelCluster UI was not used to create the cluster, it can still be used to monitor and access the cluster.

*Log on with the headnode IP address:*

The headnode IP address can be found from the EC2 console.
```bash
ssh -i /path/to/keyfile.pem ec2-user@XX.XX.XX.XX
```

CESM2 relies on high performance libraries for I/O based on HDF5 and netcdf. These files are installed by creating and then executing the script that follows. 

After logging onto the cluster headnode use your favorite editor, such as “vi” :star_struck:, to copy and paste the following script into a file called "libraryinstall.sh” placed in /home/ec2-user. 
If desired, a different editor can be installed with yum (or preferred method).

````bash
cesm_e3sm_libraryinstall.sh
#!/bin/bash
# do as sudo su

# Install the yum repo for all the oneAPI packages:
cat << EOF > /etc/yum.repos.d/oneAPI.repo
[oneAPI]
name=Intel(R) oneAPI repository
baseurl=https://yum.repos.intel.com/oneapi
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://yum.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB
EOF

# update the OS
yum -y upgrade 
yum install -y lapack-devel

####Install intel oneapi

yum -y install intel-oneapi-compiler-fortran intel-oneapi-compiler-dpcpp-cpp-and-cpp-classic intel-oneapi-mpi-devel

#removed these from compute nodes, will need environment variables in the .bashrc file
echo '/opt/ncar/software/lib' > /etc/ld.so.conf.d/ncar.conf
echo 'source /opt/intel/oneapi/setvars.sh > /dev/null' > /etc/profile.d/oneapi.sh

LIBRARY_PATH=/opt/ncar/software/lib
LD_LIBRARY_PATH=/opt/ncar/software/lib:$LD_LIBRARY_PATH
CPATH=/opt/ncar/software/include:$CPATH
FPATH=/opt/ncar/software/include:$FPATH

#fix for error messages 
# I don't think this actually fixed it
echo '-diag-disable=10441' | sudo tee -a $top_dir/intel/compiler/2021.4.0/linux/bin/intel64/icc.cfg
echo '-diag-disable=10441' | sudo tee -a $top_dir/intel/compiler/2021.4.0/linux/bin/intel64/icpc.cfg

#  build libraries
source /opt/intel/oneapi/setvars.sh --force
mkdir /tmp/sources
cd /tmp/sources
wget -q https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-1.12/hdf5-1.12.0/src/hdf5-1.12.0.tar.gz
tar zxf hdf5-1.12.0.tar.gz
cd hdf5-1.12.0
./configure --prefix=/opt/ncar/software CC=icc CXX=icpc FC=ifort
make -j 2 install
ldconfig 

cd /tmp/sources
wget -q https://downloads.unidata.ucar.edu/netcdf-c/4.9.2/netcdf-c-4.9.2.tar.gz
tar zxf netcdf-c-4.9.2.tar.gz
cd netcdf-c-4.9.2
./configure --prefix=/opt/ncar/software CC=icc CXX=icpc FC=ifort
make -j 2 install
ldconfig

cd /tmp/sources
wget -q https://downloads.unidata.ucar.edu/netcdf-fortran/4.6.0/netcdf-fortran-4.6.0.tar.gz
tar zxf netcdf-fortran-4.6.0.tar.gz
cd netcdf-fortran-4.6.0
./configure --prefix=/opt/ncar/software CC=icc CXX=icpc FC=ifort
make -j 2 install
ldconfig

cd /tmp/sources
wget -q https://parallel-netcdf.github.io/Release/pnetcdf-1.12.1.tar.gz
tar zxf pnetcdf-1.12.1.tar.gz
cd pnetcdf-1.12.1
./configure --prefix=/opt/ncar/software CC=mpicc CXX=mpicxx FC=mpiifort
make -j 2 install
ldconfig

cd /tmp

cp /usr/lib64/liblapack* /usr/lib64/libblas* /opt/ncar/software/lib
mkdir /opt/ncar/cmake
cd /tmp/sources
mkdir cmake
wget https://github.com/Kitware/CMake/releases/download/v3.22.3/cmake-3.22.3-linux-x86_64.sh
sh cmake-3.22.3-linux-x86_64.sh --prefix=/opt/ncar/cmake --skip-license

rm -rf /tmp/sources

# Change user limits for ec2-user:
cat << EOF >> /etc/security/limits.conf
ec2-user         hard    stack           -1
ec2-user         soft    stack           -1
EOF
````
Modify the file to have executable permissions:
````bash
$chmod 755 ./libraryinstall.sh
````
Run the script as su:
````bash
$sudo su
$ ./libraryinstall.sh 2>&1 | tee libraryinstall.out 
$ exit
````
Upon completion, the libraries are installed at /opt/ncar.

Next, with an editor, add the following to the end of the ec2-user .bashrc file /home/ec2-user/.bashrc. Log out of the terminal window and log back on.

````bash
module load libfabric-aws
export OMP_NUM_THREADS=1

export I_MPI_OFI_LIBRARY_INTERNAL=0
source /opt/intel/oneapi/setvars.sh --force > /dev/null
export I_MPI_FABRICS=ofi
export I_MPI_OFI_PROVIDER=efa

export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/ncar/software/lib
export PATH=/opt/ncar/software/bin:/opt/ncar/cmake/bin:.:$PATH
export I_MPI_PMI_LIBRARY=/opt/slurm/lib/libpmi.so

alias rm="rm -i"
alias cp="cp -i"
alias mv="mv -i"
````

## Step 3: Retrieve CESM2 from github

As before, the script below is copied and pasted into /home/ec2-user/cesm_install.sh using an editor. The script will 
retrieve CESM2 from github. The CESM2 branch chosen already includes hpc6a in the CESM2 machine files allowing hpc6a to be used without modifying the CESM2 files further. 

````bash
#!/bin/bash
cd /opt/ncar
git clone -b cesm2.1.4-rc.10 https://github.com/ESCOMP/CESM.git cesm
cd cesm
svn --username=guestuser --password=friendly list https://svn-ccsm-models.cgd.ucar.edu << EOF
p
yes
EOF
./manage_externals/checkout_externals

echo 'export PATH=/opt/ncar/cesm/cime/scripts:${PATH}' > /etc/profile.d/cesm.sh
echo 'export CIME_MACHINE=aws-hpc6a' >> /etc/profile.d/cesm.sh
echo 'export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/opt/ncar/software/lib' >> /etc/profile.d/cesm.sh
echo 'export NETCDF=/opt/ncar/software' >> /etc/profile.d/cesm.sh

mkdir -p /scratch/ec2-user/inputdata
chown -R ec2-user:ec2-user /scratch/ec2-user
````

Run the script as su:

````bash
$sudo su
$ ./cesm_install.sh 2>&1 | tee cesm_install.out 
$ exit
````
Log out and log back into the headnode to ensure that environment variables set properly.

### Step 4: Setup, Build and Submit a CESM2 Case

The CESM2 cluster is now ready for use! The following steps outline how to run a case. The CESM2 expert should now be right at home.

Log in as ec2-user. 
From the home directory (/home/ec2-user) create a new case with the following command:

````bash
$ create_newcase --compset QPC6 --res f19_f19_mg17 --case test_aqp1 --run-unsupported
````

Move from the /home/ec2-user directory to the newly created case directory:
````bash
$cd test_aqp1
````

Modify the job queue to the name set in the ParallelCluster config yaml file:
````bash
$ ./xmlchange JOB_QUEUE=compute --force
````

Modify the number of tasks requested for the job:

````bash
./xmlchange NTASKS=96
````

To check the current setup:
````bash
$./pelayout
````

To set the case up:
````bash
$./case.setup
$./preview_run
$./case.build
$./check_input_data –download
$./xmlchange DOUT_S=false
$./case.submit
````

The run is submitted to slurm and can be viewed in the queue with:
````bash
$squeue
````

The case status is given with:
````bash
$cat /home/ec2-user/test_aqp1/CaseStatus
````

Upon completion, the status of the run is found with:
````bash
$cat /home/ec2-user/test_aqp1/run.test_aqp1
````
The results can be found here: 
````bash
/scratch/ec2-user/test_aqp1/run
````

## Taking down the cluster and deleting AWS resources
When ready, the cluster is deleted. **Charges continue until the cluster is deleted.** :moneybag: :moneybag: :moneybag: The most significant charges come from the headnode, compute nodes (double check none are left running), and the FSx lustre /scratch volume. 

Delete and remove the cluster from the ParallelCluster cli or UI with the following command:
````bash
$pcluster delete-cluster --cluster-name cesm --region us-east-2
````

A reminder that this command is issued from the any ParallelCluster installation and not from the cluster headnode. If that installation is lost, ParallelCluster can be reinstalled, again though, not from the cluster headnode.

For advanced users, a ParallelCluster cluster can also be removed by deleting the ParallelCluster created cloudformation template from the AWS console cloudformation dashboard.

## A brief discussion on managing storage:
The AWS CLI is an effective way to move data to S3 for long term and lower cost storage. 

It is recommended that storage placed in S3 be divided into two main categories. (1) Data that may be used frequently or in the near future (such as reshaped data) and (2) data that is expected to be archived such as raw data files. By placing data of similar storage tiers into the same bucket, a bucket lifecycle rule can be easily set up and used to move the data automatically into the targeted storage tier.

### Data that may be used in the near future

Amazon S3 Intelligent-Tiering is recommended for most e3sm data. The storage tier automatically moves data to the most cost-effective S3 tier based on usage patterns and file sizes. An archival storage tier is included as one of the automated S3 Intelligent-Tiering storage tiers lowering costs for data not frequently accessed. 

### Archival data

Data that is unlikely to be retrieved in the short term can be stored in S3 Glacier Flexible Retrieval. This provides very low cost storage. The trade-off is that the data must be “restored” before retrieving it with the aws cli. Restoring the data with the “bulk” retrieval option allows for the retrieval of data without additional cost. Data needed quickly will incur an extra charge as part of the retrieval. When using S3 Glacier Flexible Retrieval, it is preferable to tar up many small files into larger files (such as the build directory) before placing that data in S3 Glacier as small files can incur small additional charges. One of the advantages of S3 Intelligent-Tiering is that this consideration is made automatically.

## Long term use of the cluster

If you choose to leave the cluster up and running monthly charges will be incurred. The significant charges include the headnode and shared storage. If left as configured, with a c6a.xlarge for a headnode, then costs will be about $275 a month when set up in Ohio and excluding compute node charges for running.

The head node can be stopped and restarted as needed (typically from the AWS console) to save on the charges resulting from the headnode instance though the storage charges remain whether data resides on those volumes or not.

## Final Cluster Thoughts 
AWS offers a free tier download of 100 GB of data from within AWS to a remote computer. Data downloaded above this amount will result in charges. Data egress charges apply both when the data is downloaded from an AWS EC2 instance using scp or downloaded from S3 directly.

The scratch drive offers plenty of room for running cases and can be made larger from the AWS console as needed. The root volume underlying the headnode can also be made larger though this space is more expensive than the /scratch volume when using FSx lustre HDD for the /scratch volume. It is noted that user /home locations are located on the root volume and if this volume accidentally fills up, one can no longer log onto the instance. It is possible to recover from this error but requires unmounting, modifying the volume, and remounting (reference). Users need to be encouraged to avoid filling up the /home volume. A follow on blog will describe how to avoid this problem.

## Conclusion and Acknowledgements
The methods described in this blog were originally put together by Brian Dobbins at NCAR. Phil Rasch refined these methods. Additional thanks goes to NCAR, NSF, and the DoE for supporting this work. SilverLining, a nonprofit organization, provides AWS support to climate researchers and contributed to this blog. SilverLining encourages and supports the use of climate codes in the cloud (the computing kind).


## References

Add CESM2 references

Intelligent Tiering
https://aws.amazon.com/s3/storage-classes/intelligent-tiering/

S3 Glacier Flexible Retrieval (is this the best reference?)
https://aws.amazon.com/about-aws/whats-new/2021/11/amazon-s3-glacier-storage-class-amazon-s3-glacier-flexible-retrieval/

Lifecycle rule
https://docs.aws.amazon.com/AmazonS3/latest/userguide/how-to-set-lifecycle-configuration-intro.html

Replacing a root volume??
https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/replace-root.html


## Appendix - How to use the ParallelCluster UI with HDD
AWS ParallelCluster now offers a companion ParallelCluster UI to simplify set up of ParallelCluster yaml config files. The yaml config file included above includes an option for Amazon FSx for Lustre with HDD storage. FSx lustre with HDD  is not yet directly supported from the UI. If the ParallelCluster UI is used, a default FSx SSD volume can be added with the UI and then yaml config file is modified when displayed to the user, right before launch to match what is used here.

In particular, tthe PerUnitStorageThroughput is modified before final deployment:

PerUnitStorageThroughput: 12

Alternatively, the cluster can be launched from the ParallelCluster UI using an SSD version of FSx lustre. If only used for a day or two, the added expense is small.


For more information about SilverLining, visit the [SilverLining](https://silverlining.ngo) website!

