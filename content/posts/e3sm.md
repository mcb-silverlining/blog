+++
title = 'Installing E3SM on AWS'
date = 2023-11-29T17:38:46-08:00
draft = false
math = true
+++

# ***This blog post is in work.***

# ***This blog post is in work.***

# ***This blog post is in work.***

How to Install and Run E3SM on AWS!

## Introduction


Installing and running the climate code E3SM on AWS

Supercomputers around the world are dedicated to the computational prediction and analysis of climate scenarios. These supercomputers run state-of-the-art models such as the DOE's E3SM, Energy Exascale Earth System Model. (Reference: [https://e3sm.org/](https://e3sm.org/)) While E3SM is available to researchers around the world, compute time on the supercomputers the codes are designed to run on, is not so easily obtained. This limits the ability of these scientists to contribute to the research needed to mitigate the effects of climate change.

Climate codes such as E3SM, CESM, and WRF are extremely compute intensive and can be complicated to install on a computing platform. AWS ParallelCluster makes setting up and running these codes easier. This blog describes how to install and run E3SM on AWS starting from AWS Parallelcluster. 


### Prerequisites & anticipated costs

This blog describes the installation and running of E3SM. As a prerequisite, an AWS account and knowledge of AWS ParallelCluster is assumed. The background for both can be found here (references,https://www.hpcworkshops.com/08-efa.html). Knowledge of E3SM is also assumed and while a small demonstration case is included, the details and complexity of running E3SM requires specific climate science expertise, references. The blog begins after the installation of AWS ParallelCluster.

The cost of working through this blog on AWS is expected to be less than $15. The cost of full-fledged runs can be estimated through the aws cost calculator by timing a short duration version of the desired run. 

The machine files included here allow E3SM to run on the AWS HPC6a instance type. The HPC6a instance type is designed specifically for HPC calculations. If you don’t have access to hpc6a, please contact AWS to make this request (where?)  


### Step 1: Modify and launch ParallelCluster with the config file 

 

AWS ParallelCluster is an AWS service that creates a cluster based on a yaml configuration file. ParallelCluster installs slurm and common HPC libraries by default. The yaml configuration file defines compute nodes, queue names, headnode size, and shared storage for the cluster. AWS ParallelCluster can be installed and run from either the users’ laptop or desktop, though often AWS cloudshell is used for a quick set up. AWS cloudshell is found in the AWS console. While the ParallelCluster UI is remarkable and available for interactively configuring the ParallelCluster yaml configuration file and then also launching it interactively, the yaml file below includes an option for FSx lustre with HDD storage. This version of FSx lustre is not yet directly supported from the UI. For this reason, installing AWS Parallelcluster and running it via the ParallelCluster CLI is recommended.

The recommended ParallelCluster configuration follows. The headnode is a c6a.xlarge (?? or do we want c6an.2xlarge? or?). The headnode needs to be large enough to support the compilation of e3sm and the required libraries. An “n” instance type is (or is not?) chosen as it offers greater bandwidth for moving large files onto the instance and from the instance to S3. The HPC6a instance is used for the compute nodes. The HPC6a is an AMD instance with 96 cores and 3XX G of memory. The instance is extremely cost effective for network intensive HPC cases. The HPC6a is only found in a few regions and availability zones. For example, Availability Zone 2 in Ohio. In the file below, replace the subnet placeholder with a subnet in one of these locations. This occurs in two spots within the configuration file.

Other modifications that will need to be made to the configuration file include substituting the name of your ssh key and double checking that the max number of compute nodes requested are within your account limits.

Two shared volumes are attached to the cluster. The first is a gp3 ebs volume which is attached at /opt/ncar and allows compute node access to the e3sm libraries. The second offers /scratch space on a shared 6T HDD FSx Lustre volume. The HDD volume is performant enough for e3sm runs while offering an excellent price point. E3SM users often require large amounts of scratch space and the FSx Lustre HDD volume can be increased from the console in increments of 6T. 

To complete this step, launch the modified yaml config file with ParallelCluster and wait for the cluster to complete.

	

Launch the modified yaml file with pcluster and wait for the cluster creation to complete (approximately 20 minutes &lt;- check this).

$pcluster create-cluster —cluster-configuration e3sm-blog.yaml —cluster-name blogcluster —region us-east-2


### Step 2: Install the libraries required by E3SM 

	

The E3SM machine files target a specific compiler and mpi configuration. It also relies on high performance libraries for I/O based on hdf5 and netcdf. These files are installed with the script that follows. 

Start by logging into the cluster as ec2-user using ssh. Copy and paste the following script into a file called “e3sm_library.sh” placed in /home/ec2-user. 

Modify the file to have executable permissions:

$chmod 755 ./e3sm_library.sh

Run the script as su:

$sudo su

$ ./e3sm_libraryinstall.sh 2>&1 | tee e3sm_libraryinstall.out 

$ exit

Upon completion, libraries are installed at /opt/ncar.

Using your favorite editor, add the following to the end of the ec2-user .bashrc file. Log out of the terminal window and log back on.


### Step 3: Bring over the E3SM files

E3SM documentation is found at the E3SM confluence wiki:

[https://acme-climate.atlassian.net/wiki/spaces/DOC/overview?homepageId=1931641291](https://acme-climate.atlassian.net/wiki/spaces/DOC/overview?homepageId=1931641291)

As stated on the e3sm confluence page:

_"Installing" is a bit of a misnomer.  One does not install E3SM like system software: in a location such as  /usr/local/bin for everyone to use.   Instead, a user creates executables for their cases and keeps them in their own directories._

Reference: [https://acme-climate.atlassian.net/wiki/spaces/DOC/pages/7996898/Installation%2BGuide](https://acme-climate.atlassian.net/wiki/spaces/DOC/pages/7996898/Installation%2BGuide)

Those familiar with running E3SM will recognize the e3sm.sh run script that guides e3sm usage. This file includes specific case and running information and serves to document the run conditions of a case. The example below is set up to run the e3sm maintenance version 2.0. A water cycle (???) case will be run (what gets said here?) 

As the e3sm code will be brought over from github, a github account is required with an added ssh key to that account. Documentation on how to set that up is found here. 

https://docs.github.com/en/authentication/connecting-to-github-with-ssh

https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account

This key can be different from the key that is on the newly created cluster however the key added to github account needs to be added to the cluster to allow the e3sm.sh script to clone the e3sm repo. Place the key at /home/ec2-user/.ssh location via scp or by using a copy and paste with an editor. Once the key is located, modify the permissions:

$chmod 600 ~/.ssh/githubkey.pem

The key is then added to the ec2-user account as follows:

$eval $(ssh-agent); ssh-add -k ~/.ssh/githubkey.pem

The e3sm run script (e3sm.sh following) is used to download, set up, and build e3sm for each case. A description of this process is given here if more details are required:

https://acme-climate.atlassian.net/wiki/spaces/DOC/pages/2309226536/Running+E3SM+step-by-step+guide#Running-short-tests

Place the file, e3sm.sh in the /home/ec2-user directory and change the permissions to executable:

$chmod 755 ./e3sm.sh

Do we need to fix any of the names or items in the e3sm.sh file? Linda to ask Phil. Ask Phil about the RUN_TYPE=startup modification I made. Should this be outside the script?

Some modifications to this file from the standard template include increasing the default wall time (as this run is on only 96 cores by default), modifying to a test case from a production run. While the flag “do_fetch_code” has been changed to true. The other procedure flags are set to false:

do_fetch_code=true

do_create_newcase=false

do_case_setup=false

do_case_build=false

do_case_submit=false

Launch the e3sm.sh script to “fetch the code” from github:

$./e3sm.sh 2>&1 | tee e3sm.fetch.out

During the run a prompt to use the github key will require an answer of “yes”.

When the script is complete, e3sm maintenance version 2 will be checked out to this location /home/ec2-user/E3SMv2_AWS/code/maint2.0

These files do not yet include the HPC6a machine files.


### Step 4: Add the HPC6a machine files to the e3sm files

The following three files are added to the e3sm files just downloaded. The first two files:

config_machines.xml

config_compilers.xml

are placed in this location:

/home/ec2-user/E3SMv2_AWS/code/maint2.0/cime_config/machines 

Note that these files are intended to replace the existing files of the same name.

Place the third file:

intel_aws-hpc6a.cmake

at this location:

/home/ec2-user/E3SMv2_AWS/code/maint2.0/cime_config/machines/cmake_macros

This file does not replace an existing file but is added at this location.


### Step 5: Setup and Build an E3SM case

With the machine files updated, the e3sm.sh script is edited to setup and run the case. Edit the e3sm.sh script and update the following values: 

do_fetch_code=false

do_create_newcase=true

do_case_setup=true

do_case_build=true

The last action remains as false for now:

do_case_submit=false

Rerun the e3sm.sh script:

$./e3sm.sh 2>&1 | tee e3sm.setup&build.out

The script will use the newly added machine files to setup and build the necessary executables. Once complete, the new case is found here:

/scratch/geostrat/E3SMv2/e3sm_blog_test/tests/L_1x1_nmonths/

Move to this location:

/scratch/geostrat/E3SMv2/e3sm_blog_test/tests/L_1x1_nmonths/case_scripts

to double check that all the necessary input data has been downloaded:

$check_input_data —download 

If this command is not successful, the command must be rerun until it is. Occasionally this means waiting until the next morning for the files to become available. Upon successful completion the case is ready to run. 


### Step 6: Run the E3SM case

The case is now ready to run. Moving back to the home directory, the e3sm.sh script is edited and the case actions are changed as follows:

do_fetch_code=false

do_create_newcase=false

do_case_setup=false

do_case_build=false

do_case_submit=true

The e3sm.sh script is run once more: 

$./e3sm.sh 2>&1 | tee e3sm.run.out

After successful completion of the script, the case is launched to slurm and can be seen with squeue command:

$squeue

The job will take about three hours to complete.


### Taking down the cluster and deleting AWS resources

When ready, the E3SM cluster can be removed and all charges will be stopped with the ParallelCluster cli. From the installation of ParallelCluster:

$pcluster delete-cluster —cluster-name blogcluster —region us-east-2


### What’s next?

If you choose to leave the cluster up and running monthly charges will be incurred. The significant charges include the headnode and shared storage. If left as configured (c6a.xl) then costs will be on the order of $275 a month (excluding compute node charges for running).

The head node can be stopped and restarted to save on the headnode instance charges.

 

Simulation data created can be stored in S3 for longer term and lower cost storage. The simulation data can also be downloaded. For data download, it is recommended that the data be transferred to S3 first and downloaded from there. Downloading large data files via scp can incur more charges than a download directly from S3. Consult the aws cost calculator (calculator.aws) for AWS costs.


### Conclusion and Acknowledgements

AWS offers a means for all research organizations access the computing resources necessary to perform this important work. The methods described in this blog were originally put together by Brian Dobbins at NCAR. Thanks goes to Brian and these organizations for supporting this work. Additional thanks goes to NCAR, NSF, and the DoE for supporting this work. SilverLining, a nonprofit organization, contributed to this blog. SilverLining encourages and supports the use of climate codes in the cloud (the computing kind).

	
AAAA
# Installing and Running E3SM with AWS ParallelCluster

This gist demonstrates how to install E3SM to AWS ParallelCluster.

## What is AWS ParallelCluster?
AWS ParallelCluster is an AWS service that creates a cluster based on a yaml configuration file. The yaml configuration file defines compute nodes, queue names, headnode size, and shared storage for the cluster. AWS ParallelCluster is a python script that is installed and run from either the users’ laptop, desktop, or from AWS via AWS cloudshell, cloud9, or an ec2 instance. AWS ParallelCluster can be run from the ParallelCluster UI though as of this writing, the UI does not support the version of AWS FSx Lustre used below.

## Prerequisites
* Knowledge of AWS ParallelCluster with ParallelCluster installed and available to deploy a cluster based on the supplied yaml configuraiton file. The background for both can be found here references,https://www.hpcworkshops.com/08-efa.html).
* Knowledge of E3SM is also assumed though a small demonstration case is included.
* An AWS SSH key is required to launch the cluster and can be created from the Amazon EC2 dashboard if one is not already available.
* The procedure described here uses the AWS HPC6a instance type (https://aws.amazon.com/ec2/instance-types/hpc6/). The HPC6a instance type is designed specifically for HPC calculations. Double check AWS limits on the AWS Service quota panel from the AWS console. The hpc6a limit is found as part of the “Running On-Demand HPC instances” quota name from the Amazon Elastic Cloud Compute dashboard card. Request a limit increase if necessary before proceeding.

## Anticipated Costs
The AWS costs for working through this gist is expected to be less than $15. The cost of full-fledged runs can be estimated through the aws cost calculator by timing a short duration of the desired run and extrapolating to the full time desired.

## The Cluster
The steps below walk the user through launching a cluster, installing required libraries, fetching the E3SM, and launching a case.

### Step 1: Create a cluster with the following yaml config file
Copy the following parallel cluster yaml config file to where AWS ParallelCluster is installed. Make the following edits:
* replace the subnet placeholder with a subnet where HPC6a is available, us-east-2, AZ2 is often used. The subnet name that corresponds to us-east-2, AZ2 can be found in the VPC section of the AWS console. The subnet needs to be updated in two spots within the configuration file.
* Substitute the name of your ssh key
* Double check that the max number of compute nodes requested are within your account limits. The cluster will fail if the hpc6a account limits are exceeded.

Two shared volumes are designated to be attached to the cluster. The first is an Amazon EBS volume (general purpose SSD-based) which is attached at /opt/ncar and allows compute node access E3SM and the libraries. The second an Amazon FSx for Lustre volume offers /scratch space on a shared 6TB PERSISTENT 1 HDD. The HDD volume is performant enough for E3SM runs while offering an excellent price point. E3SM users often require large amounts of scratch space and the Amazon FSx for Lustre volume can be increased from the console in increments of 6TB.

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





### Step 2: Install the libraries

E3SM relies on high performance libraries for I/O based on HDF5 and netcdf. These files are installed by creating, then executing the script that follows.

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

```bash
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
```

Step 3: Retrieve E3SM from github

In this blog, we supply a bash shell script file called “e3sm.sh”. The e3sm.sh run script serves to document the run conditions of the case. The example here is set up to run the E3SM maintenance version 2.0. The script is based on a template provided with E3SM called run_e3sm.template.sh but with a few modifications to adapt it to the AWS environment. 

In general, the script is modified to suit your needs. You can create the model executable, acquire needed datasets, then submit and run the executable to perform a simulation. 

The e3sm.sh script capable of: 
specifying a model configuration (e.g., the model resolution, and the components (atmosphere, ocean, ice, etc) intended to be used in the simulations, 
acquiring (downloading, and checking out) the model’s (more than a million lines of ) source code from the E3SM git repository, 
then automatically configuring the model to 
subsequently download additional files (specifying boundary conditions, initial conditions, and ancillary files) needed for the model to run that configuration,
Execute commands to compile and build the model executable to run the E3SM for that configuration,
More commands to submit, and resubmit (via slurm) a sequence of jobs that run a simulation for a specified period, and continue to extend that simulation by resubmitting job for a given number of years. 
The script also supports moving the files that are written during the simulation to a specified location in preparation for subsequent analysis, or for transfer to an S3 bucket for long term archival.

While the script can be configured to perform all these steps with one execution of the script, it is frequently the case that it is executed multiple times, each time performing another step in the sequence.  For more information, please refer to the following reference. 

https://acme-climate.atlassian.net/wiki/spaces/DOC/pages/2309226536/Running+E3SM+step-by-step+guide#Running-short-tests

A github account is required to bring over the E3SM code.That account needs an added ssh key for authentication. Documentation on how to set that up is provided here: 

https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account

This key can be different from the key that used in the ParallelCluter config file however the key added to github account needs to be added to the cluster to allow the e3sm.sh script to clone the e3sm repo. Place the key at /home/ec2-user/.ssh location via scp or by using a copy and paste with an editor. Once the key is located, modify the permissions on the key file:

$chmod 600 ~/.ssh/githubkey.pem

The key is then added to the ec2-user account as follows:

$eval $(ssh-agent); ssh-add -k ~/.ssh/githubkey.pem

Place the file, e3sm.sh in the /home/ec2-user directory and change the permissions to executable:

$chmod 755 ./e3sm.sh

If desired, edit the e3sm.sh and change the CASE_NAME if desired. It is currently set as follows:
readonly CASE_NAME="test_case_for_blog"

While viewing the script, note that the flag “do_fetch_code” is set to true. The other procedure flags are set to false:

do_fetch_code=true
do_create_newcase=false
do_case_setup=false
do_case_build=false
do_case_submit=false

Launch the e3sm.sh script to “fetch the code” from github:

$./e3sm.sh 2>&1 | tee e3sm.fetch.out

During the run a prompt to use the github key will require an answer of “yes”.

When the script is complete, e3sm maintenance version 2 will be checked out to this location /home/ec2-user/E3SMv2/code/maint-2.0

These files do not yet include the HPC6a machine files.
[E3SM] Step 4: Add the HPC6a machine files to the E3SM files
The following three files are added to the e3sm code just downloaded. The first two files:

config_machines.xml
config_compilers.xml

are placed in this location:

/home/ec2-user/E3SMv2/code/maint-2.0/cime_config/machines 

Note that these files are intended to entirely replace the existing files of the same name.

Place the third file:

intel_aws-hpc6a.cmake

at this location:

/home/ec2-user/E3SMv2/code/maint-2.0/cime_config/machines/cmake_macros

This file is an addition and does not replace an existing file.
[CESM] Step 5: Setup, Build and Submit a CESM Case

The steps are all functionally here but a few more words as helpful to this section

From the cluster, log out and log back in to ensure that environment variables are picked up.

As ec2-user from the /home/ec2-user create a new case:

(Note that these are all double dashes and the formatting did not come through.)
$create_newcase –compset QPC6 –res f19_f19_mg17 –case test_aqp1 –run-unsupported

Move from the /home/ec2-user directory to the newly created case directory:
$cd test_aqp1

Modify the job queue to the name set in the ParallelCluster config yaml file:
$./xmlchange JOB_QUEUE=compute --force

Modify the number of tasks requested for the job:

./xmlchange NTASKS=96

To check the current setup:

$./pelayout

Set the case up:

$./case.setup

./preview_run
./case.build
./check_input_data –download
./xmlchange DOUT_S=false
./case.submit
The run is submitted to slurm and can be seen in the queue by issuing 
$squeue

The case status is found here:
$cat /home/ec2-user/test_aqp1/CaseStatus

Upon completion, the status of the run is found with:

$cat /home/ec2-user/test_aqp1/run.test_aqp1

The results can be found here (are these the results?)
/scratch/ec2-user/test_aqp1/run





[E3SM] Step 5: Setup, Build and Submit an E3SM Case

With the machine files updated, the e3sm.sh script is edited to create the new case, set up the files, build the code, and submit the run. Edit the e3sm.sh script and update the following values: 

do_fetch_code=false
do_create_newcase=true
do_case_setup=true
do_case_build=true
do_case_submit=true

Rerun the e3sm.sh script:
$./e3sm.sh 2>&1 | tee e3sm.new_setup_build_submit.out

The script will use the newly added machine files to setup and build the necessary executables. Once complete, the new case is found here:

/scratch/${USER}/E3SMv2/test_case_for_blog/tests/XS_2x5_ndays/

After successful completion of the script, the case is launched to slurm and can be seen with squeue command:

$squeue

The job will take about one hour to complete on one hpc6a compute node.













# Step 3: Retrieve E3SM from github

In this blog, we supply a bash shell script file called “e3sm.sh”. The e3sm.sh run script serves to document the run conditions of the case. The example here is set up to run the E3SM maintenance version 2.0. The script is based on a template provided with E3SM called run_e3sm.template.sh but with a few modifications to adapt it to the AWS environment. 

In general, the script is modified to suit your needs. You can create the model executable, acquire needed datasets, then submit and run the executable to perform a simulation. 

The e3sm.sh script capable of: 
specifying a model configuration (e.g., the model resolution, and the components (atmosphere, ocean, ice, etc) intended to be used in the simulations, 
acquiring (downloading, and checking out) the model’s (more than a million lines of ) source code from the E3SM git repository, 
then automatically configuring the model to 
subsequently download additional files (specifying boundary conditions, initial conditions, and ancillary files) needed for the model to run that configuration,
Execute commands to compile and build the model executable to run the E3SM for that configuration,
More commands to submit, and resubmit (via slurm) a sequence of jobs that run a simulation for a specified period, and continue to extend that simulation by resubmitting job for a given number of years. 
The script also supports moving the files that are written during the simulation to a specified location in preparation for subsequent analysis, or for transfer to an S3 bucket for long term archival.

While the script can be configured to perform all these steps with one execution of the script, it is frequently the case that it is executed multiple times, each time performing another step in the sequence.  For more information, please refer to the following reference. 

https://acme-climate.atlassian.net/wiki/spaces/DOC/pages/2309226536/Running+E3SM+step-by-step+guide#Running-short-tests

A github account is required to bring over the E3SM code.That account needs an added ssh key for authentication. Documentation on how to set that up is provided here: 

https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account

This key can be different from the key that used in the ParallelCluter config file however the key added to github account needs to be added to the cluster to allow the e3sm.sh script to clone the e3sm repo. Place the key at /home/ec2-user/.ssh location via scp or by using a copy and paste with an editor. Once the key is located, modify the permissions on the key file:

$chmod 600 ~/.ssh/githubkey.pem

The key is then added to the ec2-user account as follows:

$eval $(ssh-agent); ssh-add -k ~/.ssh/githubkey.pem

Place the file, e3sm.sh in the /home/ec2-user directory and change the permissions to executable:

$chmod 755 ./e3sm.sh

If desired, edit the e3sm.sh and change the CASE_NAME if desired. It is currently set as follows:
readonly CASE_NAME="test_case_for_blog"

While viewing the script, note that the flag “do_fetch_code” is set to true. The other procedure flags are set to false:

do_fetch_code=true
do_create_newcase=false
do_case_setup=false
do_case_build=false
do_case_submit=false

Launch the e3sm.sh script to “fetch the code” from github:

$./e3sm.sh 2>&1 | tee e3sm.fetch.out

During the run a prompt to use the github key will require an answer of “yes”.

When the script is complete, e3sm maintenance version 2 will be checked out to this location /home/ec2-user/E3SMv2/code/maint-2.0

These files do not yet include the HPC6a machine files.
[E3SM] Step 4: Add the HPC6a machine files to the E3SM files
The following three files are added to the e3sm code just downloaded. The first two files:

config_machines.xml
config_compilers.xml

are placed in this location:

/home/ec2-user/E3SMv2/code/maint-2.0/cime_config/machines 

Note that these files are intended to entirely replace the existing files of the same name.

Place the third file:

intel_aws-hpc6a.cmake

at this location:

/home/ec2-user/E3SMv2/code/maint-2.0/cime_config/machines/cmake_macros

This file is an addition and does not replace an existing file.
[CESM] Step 5: Setup, Build and Submit a CESM Case

The steps are all functionally here but a few more words as helpful to this section

From the cluster, log out and log back in to ensure that environment variables are picked up.

As ec2-user from the /home/ec2-user create a new case:

(Note that these are all double dashes and the formatting did not come through.)
$create_newcase –compset QPC6 –res f19_f19_mg17 –case test_aqp1 –run-unsupported

Move from the /home/ec2-user directory to the newly created case directory:
$cd test_aqp1

Modify the job queue to the name set in the ParallelCluster config yaml file:
$./xmlchange JOB_QUEUE=compute --force

Modify the number of tasks requested for the job:

./xmlchange NTASKS=96

To check the current setup:

$./pelayout

Set the case up:

$./case.setup

./preview_run
./case.build
./check_input_data –download
./xmlchange DOUT_S=false
./case.submit
The run is submitted to slurm and can be seen in the queue by issuing 
$squeue

The case status is found here:
$cat /home/ec2-user/test_aqp1/CaseStatus

Upon completion, the status of the run is found with:

$cat /home/ec2-user/test_aqp1/run.test_aqp1

The results can be found here (are these the results?)
/scratch/ec2-user/test_aqp1/run





[E3SM] Step 5: Setup, Build and Submit an E3SM Case

With the machine files updated, the e3sm.sh script is edited to create the new case, set up the files, build the code, and submit the run. Edit the e3sm.sh script and update the following values: 

do_fetch_code=false
do_create_newcase=true
do_case_setup=true
do_case_build=true
do_case_submit=true

Rerun the e3sm.sh script:
$./e3sm.sh 2>&1 | tee e3sm.new_setup_build_submit.out

The script will use the newly added machine files to setup and build the necessary executables. Once complete, the new case is found here:

/scratch/${USER}/E3SMv2/test_case_for_blog/tests/XS_2x5_ndays/

After successful completion of the script, the case is launched to slurm and can be seen with squeue command:

$squeue

The job will take about one hour to complete on one hpc6a compute node.
Taking down the cluster and deleting AWS resources
When ready, the cluster can be removed and all charges will be stopped. Delete and remove the cluster with the ParallelCluster cli. From the location of the ParallelCluster installation (for example, the laptop or cloud shell terminal used to launch the cluster), issue the following command:

$pcluster delete-cluster —cluster-name blogcluster —region us-east-2

A ParallelCluster cluster can also be removed by deleting the cloudformation template that launched the cluster from the AWS console cloud formation dashboard.
A discussion on managing storage:
This blog would be incomplete without addressing recommended best practices for managing the large amount of data created by the earth system model runs. Simulation data created can be stored in S3 for longer term and lower cost storage. 

While the /scratch volume provided by the FSx lustre volume is persistent and economical as set up in this blog, S3 offers storage tiers that are even more cost effective. The AWS CLI is an effective way to move data from the cluster to S3. 

It is recommended that storage placed in S3 be divided into two main categories. (1) Data that may be used frequently or in the near future (such as reshaped data) and (2) data that is expected to be archived such as raw data files. By placing data of similar storage tiers into the same bucket, a bucket lifecycle rule can be easily set up and used to move the data automatically into the targeted storage tier.

Data that may be used in the near future

Amazon S3 Intelligent-Tiering is recommended for most e3sm data. The storage tier automatically moves data to the most cost-effective S3 tier based on usage patterns and file sizes. An archival storage tier is included as one of the automated S3 Intelligent-Tiering storage tiers lowering costs for data not frequently accessed. 

Archival data

Data that is unlikely to be retrieved in the short term can be stored in S3 Glacier Flexible Retrieval. This provides very low cost storage. The trade-off is that the data must be “restored” before retrieving it with the aws cli. Restoring the data with the “bulk” retrieval option allows for the retrieval of data without additional cost. Data needed quickly will incur an extra charge as part of the retrieval. When using S3 Glacier Flexible Retrieval, it is preferable to tar up many small files into larger files (such as the build directory) before placing that data in S3 Glacier as small files can incur small additional charges. One of the advantages of S3 Intelligent-Tiering is that this consideration is made automatically.
What’s next?
If you choose to leave the cluster up and running monthly charges will be incurred. The significant charges include the headnode and shared storage. If left as configured, with an c6a.xlarge for a headnode, then costs will be about $275 a month when set up in Ohio and excluding compute node charges for running.

Don’t think this will be needed:
https://calculator.aws/#/estimate?id=cdf026260585f55e8bf9ceaf0436a2dd2cf4fb00

The head node can be stopped and restarted as needed (typically from the AWS console) to save on the charges resulting from the headnode instance.
Final Cluster Thoughts 
AWS offers a free tier download of 100 GB of data from within AWS to a remote computer. Data downloaded above this amount will result in charges. Data egress charges apply both when the data is downloaded from an AWS EC2 instance using scp or downloaded from S3 directly.

The scratch drive offers plenty of room for running cases and can be made larger from the AWS console as needed. The root volume underlying the headnode can also be made larger though this space is more expensive than the /scratch volume when using FSx lustre HDD for the /scratch volume. It is noted that user /home locations are located on the root volume and if this volume accidentally fills up, one can no longer log onto the instance. It is possible to recover from this error but requires unmounting, modifying the volume, and remounting (reference). Users need to be encouraged to avoid filling up the /home volume.
Conclusion and Acknowledgements
AWS provides research organizations with the computing resources necessary to perform the important work of understanding our climate. The methods described in this blog were originally put together by Brian Dobbins at NCAR. Phil Raush refined these methods and coauthored this blog. Additional thanks goes to NCAR, NSF, and the DoE for supporting this work. SilverLining, a nonprofit organization, provides AWS support to climate researchers and contributed to this blog. SilverLining encourages and supports the use of climate codes in the cloud (the computing kind).


References

Add CESM references

E3SM documentation is found at the E3SM confluence wiki:
https://acme-climate.atlassian.net/wiki/spaces/DOC/overview?homepageId=1931641291

The run_e3sm.template.sh file can be found here:
https://github.com/E3SM-Project/E3SM/blob/master/run_e3sm.template.sh

Intelligent Tiering
https://aws.amazon.com/s3/storage-classes/intelligent-tiering/

S3 Glacier Flexible Retrieval (is this the best reference?)
https://aws.amazon.com/about-aws/whats-new/2021/11/amazon-s3-glacier-storage-class-amazon-s3-glacier-flexible-retrieval/

Lifecycle rule
https://docs.aws.amazon.com/AmazonS3/latest/userguide/how-to-set-lifecycle-configuration-intro.html

Replacing a root volume??
https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/replace-root.html



AAAA
 

For more information about SilverLining, visit the [SilverLining](https://silverlining.ngo) website!

