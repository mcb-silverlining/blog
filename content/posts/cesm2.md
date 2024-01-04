+++
title = 'Installing CESM2 to AWS'
date = 2023-11-29T17:44:22-08:00
draft = false
+++
## Introduction

This blog post is in work.

How to install CESM2 to *AWS*!

# Installing and Running CESM with AWS ParallelCluster

This gist demonstrates how to install CESM to AWS ParallelCluster.

## What is AWS ParallelCluster? 
AWS ParallelCluster is an AWS service that creates a cluster based on a yaml configuration file. The yaml configuration file defines compute nodes, queue names, headnode size, and shared storage for the cluster. AWS ParallelCluster is a python script that is installed and run from either the users’ laptop, desktop, or from AWS via AWS cloudshell, cloud9, or an ec2 instance. AWS ParallelCluster can be run from the ParallelCluster UI though as of this writing, the UI does not support the version of AWS FSx Lustre used below. 

## Prerequisites 
* Knowledge of AWS ParallelCluster with ParallelCluster installed and available to deploy a cluster based on the supplied yaml configuraiton file. The background for both can be found here references,https://www.hpcworkshops.com/08-efa.html). 
* Knowledge of CESM is also assumed though a small demonstration case is included. 
* An AWS SSH key is required to launch the cluster and can be created from the Amazon EC2 dashboard if one is not already available. 
* The procedure described here uses the AWS HPC6a instance type (https://aws.amazon.com/ec2/instance-types/hpc6/). The HPC6a instance type is designed specifically for HPC calculations. Double check AWS limits on the AWS Service quota panel from the AWS console. The hpc6a limit is found as part of the “Running On-Demand HPC instances” quota name from the Amazon Elastic Cloud Compute dashboard card. Request a limit increase if necessary before proceeding.

## Anticipated Costs
The AWS costs for working through this gist is expected to be less than $15. The cost of full-fledged runs can be estimated through the aws cost calculator by timing a short duration of the desired run and extrapolating to the full time desired. 

## The Cluster 
The steps below walk the user through launching a cluster, installing required libraries, fetching the CESM, and launching a case. 

### Step 1: Create a cluster with the following yaml config file 
Copy the following parallel cluster yaml config file to where AWS ParallelCluster is installed. Make the following edits:
* replace the subnet placeholder with a subnet where HPC6a is available, us-east-2, AZ2 is often used. The subnet name that corresponds to us-east-2, AZ2 can be found in the VPC section of the AWS console. The subnet needs to be updated in two spots within the configuration file.
* Substitute the name of your ssh key 
* Double check that the max number of compute nodes requested are within your account limits. The cluster will fail if the hpc6a account limits are exceeded.

Two shared volumes are designated to be attached to the cluster. The first is an Amazon EBS volume (general purpose SSD-based) which is attached at /opt/ncar and allows compute node access CESM and the libraries. The second an Amazon FSx for Lustre volume offers /scratch space on a shared 6TB PERSISTENT 1 HDD. The HDD volume is performant enough for CESM runs while offering an excellent price point. CESM users often require large amounts of scratch space and the Amazon FSx for Lustre volume can be increased from the console in increments of 6TB. 

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

To complete this step, launch the modified yaml config file with ParallelCluster and wait for the cluster creation to complete, approximately 15 minutes.

```bash
$ pcluster create-cluster —cluster-configuration cesm.yaml —cluster-name cesm —region us-east-2
```

### Step 2: Install the libraries
	
CESM relies on high performance libraries for I/O based on HDF5 and netcdf. These files are installed by creating, then executing the script that follows. 

Start by logging into the cluster as ec2-user using ssh. Using your favorite editor, such as “vi”, copy and paste the following script into a file called "libraryinstall.sh” placed in /home/ec2-user. 

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

Next, add the following to the end of the ec2-user .bashrc file. Log out of the terminal window and log back on.

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

## Step 3: Retrieve CESM from github
Retrieve CESM from github with the following script below called cesm_install.sh. This CESM branch chosen already includes hpc6a in the CESM machine files allowing hpc6a to be used without modifying the CESM files further. Copy and paste the following into a file called "cesm_install.sh".

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

### Step 4: Setup, Build and Submit a CESM Case

The steps are all functionally here but a few more words as helpful to this section

From the cluster, log out and log back in to ensure that environment variables are picked up.

As ec2-user from the /home/ec2-user create a new case:

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

Set the case up:
# Need more words in the following
````bash
$./case.setup

$./preview_run
$./case.build
$./check_input_data –download
$./xmlchange DOUT_S=false
$./case.submit
````

The run is submitted to slurm and can be seen in the queue by issuing 
````bash
$squeue
````

The case status is found here:
````bash
$cat /home/ec2-user/test_aqp1/CaseStatus
````

Upon completion, the status of the run is found with:
````bash
$cat /home/ec2-user/test_aqp1/run.test_aqp1
````
The results can be found here (are these the results?)
````bash
/scratch/ec2-user/test_aqp1/run
````

## Taking down the cluster and deleting AWS resources
When ready, the cluster can be removed and all charges will be stopped. Delete and remove the cluster with the ParallelCluster cli. From the location of the ParallelCluster installation (for example, the laptop or cloud shell terminal used to launch the cluster), issue the following command:
````bash
check that these are double dashses (and also throught this gist)
$pcluster delete-cluster --cluster-name cesm --region us-east-2
````

A ParallelCluster cluster can also be removed by deleting the cloudformation template that launched the cluster from the AWS console cloud formation dashboard.

## A discussion on managing storage:
The AWS CLI is an effective way to move data to S3 for long term and lwoer cost storage. 

It is recommended that storage placed in S3 be divided into two main categories. (1) Data that may be used frequently or in the near future (such as reshaped data) and (2) data that is expected to be archived such as raw data files. By placing data of similar storage tiers into the same bucket, a bucket lifecycle rule can be easily set up and used to move the data automatically into the targeted storage tier.

### Data that may be used in the near future

Amazon S3 Intelligent-Tiering is recommended for most e3sm data. The storage tier automatically moves data to the most cost-effective S3 tier based on usage patterns and file sizes. An archival storage tier is included as one of the automated S3 Intelligent-Tiering storage tiers lowering costs for data not frequently accessed. 

### Archival data

Data that is unlikely to be retrieved in the short term can be stored in S3 Glacier Flexible Retrieval. This provides very low cost storage. The trade-off is that the data must be “restored” before retrieving it with the aws cli. Restoring the data with the “bulk” retrieval option allows for the retrieval of data without additional cost. Data needed quickly will incur an extra charge as part of the retrieval. When using S3 Glacier Flexible Retrieval, it is preferable to tar up many small files into larger files (such as the build directory) before placing that data in S3 Glacier as small files can incur small additional charges. One of the advantages of S3 Intelligent-Tiering is that this consideration is made automatically.

## Long term use of the cluster

If you choose to leave the cluster up and running monthly charges will be incurred. The significant charges include the headnode and shared storage. If left as configured, with an c6a.xlarge for a headnode, then costs will be about $275 a month when set up in Ohio and excluding compute node charges for running.

The head node can be stopped and restarted as needed (typically from the AWS console) to save on the charges resulting from the headnode instance.

## Final Cluster Thoughts 
AWS offers a free tier download of 100 GB of data from within AWS to a remote computer. Data downloaded above this amount will result in charges. Data egress charges apply both when the data is downloaded from an AWS EC2 instance using scp or downloaded from S3 directly.

The scratch drive offers plenty of room for running cases and can be made larger from the AWS console as needed. The root volume underlying the headnode can also be made larger though this space is more expensive than the /scratch volume when using FSx lustre HDD for the /scratch volume. It is noted that user /home locations are located on the root volume and if this volume accidentally fills up, one can no longer log onto the instance. It is possible to recover from this error but requires unmounting, modifying the volume, and remounting (reference). Users need to be encouraged to avoid filling up the /home volume.

## Conclusion and Acknowledgements
The methods described in this blog were originally put together by Brian Dobbins at NCAR. Phil Raush refined these methods. Additional thanks goes to NCAR, NSF, and the DoE for supporting this work. SilverLining, a nonprofit organization, provides AWS support to climate researchers and contributed to this blog. SilverLining encourages and supports the use of climate codes in the cloud (the computing kind).


## References

Add CESM references

Intelligent Tiering
https://aws.amazon.com/s3/storage-classes/intelligent-tiering/

S3 Glacier Flexible Retrieval (is this the best reference?)
https://aws.amazon.com/about-aws/whats-new/2021/11/amazon-s3-glacier-storage-class-amazon-s3-glacier-flexible-retrieval/

Lifecycle rule
https://docs.aws.amazon.com/AmazonS3/latest/userguide/how-to-set-lifecycle-configuration-intro.html

Replacing a root volume??
https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/replace-root.html




For more information about SilverLining, visit the [SilverLining](https://silverlining.ngo) website!

