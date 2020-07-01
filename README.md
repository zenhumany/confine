# Confine

This framework can help in debloating containers. This is a work in progress 
and each module has been described separately below. The general goal is to 
remove functionalities not required by containers by performing static 
analysis.

## Prerequisites

This project has only been tested on Ubuntu 16.04. Due to usage of specific
debian-based tools (such as dpkg and apt-file) use on other operating systems
at your own risk.
All the scripts have been written in coordinance with python version 3.7.
```
sudo apt update
sudo apt install -y python3.7
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo apt install -y sysdig
```

In case your system does not have the python3.7 in the default repositories, you
may need to add them through third party PPAs separately.
An example is provided below:
```
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
sudo apt install -y python3.7
```
***NOTE: Adding untrusted PPAs to your system repository is not advised. It can lead 
to the installation of malicious applications. Use third party PPAs at your own risk.***

In case enabling the docker service fails due to the following error:
```
Failed to start docker.service: Unit docker.service is masked
```

You can run the following command before starting and enabling the service.
```
sudo systemctl unmask docker.service
```


## Installation
You must run the following commands before running the main application.
```
git submodule update --init --recursive
./initialize.sh
```

## Call Function Graph Extraction
We have used an LLVM pass to create a call function graph for musl-libc 
which maps all the exported functions to system calls. We also used the 
gcc RTL and the egypt tool to create a cfg for glibc.

## Container Profile Creation
The main script in this repo is the createProfiles.py file which uses 
previously created CFGs for musl-libc and glibc, along with a list of 
images and creates respective SECCOMP profiles for each.
-l: glibc callgraph
-m: musl-libc callgraph
-f: glibc shared object
-n: musl-libc shared object
-i: input file containing JSON of images
-o: path to store binaries and libraries extracted from container
-p: path to Docker default seccomp profile
-r: path to store results (seccomp profiles created)
-g: path to special cases containers like golang ones
-c: path to other CFGs, in case there are any other libraries with CFGs 
-d: debugging enabled or disabled
--finegrained: [Optional] Passing this argument enables the fine grained policy generation.
--allbinaries: [Optional] Passing this argument causes the extraction of all 
binaries instead of only the ones run during the 30 seconds. This would cause 
and extremely more conservative filter.


```
python3.7 createProfiles.py -l libc-callgraphs/glibc.callgraph -m libc-callgraphs/musllibc.callgraph -i images.json -o output/ -p default.seccomp.json -r results/ -g go.syscalls/ -c otherCfgs/ 
```

-i: The input file is in the JSON format. The images.json provided in the 
contains the images used for our paper. You can add or remove images as you 
need and see fit. The format for each image is as shown below::
```
"nginx": {
        "enable": "false",
        "image-name": "nginx",
        "image-url": "nginx",
        "category": [
            "ApplicationInfrastructure"
        ],
        "pull-count": 1960017377,
        "official": true,
        "options": "",
        "args": "",
        "dependencies": {},
        "id": "15"
    }
```
-g: If the container uses languages such as Golang and we have extracted system 
calls which are required through a different mechanism such as CFG extraction 
at the source code level, we can create a file named [imagename].syscalls and 
place it in the path specified by -g.

-c: In cases there might be libraries which also provide wrappers for system 
calls we can create the CFGs for these libraries as well and place them in 
the folder specicified by this option. These CFGs will also be used in case 
the optional --finegrained option is enabled and use them to create stricter 
syscall policies.

## CVE Mapper
In the evaluation section of our [paper](https://www3.cs.stonybrook.edu/~sghavamnia/papers/confine.raid20.pdf) 
we have shown how hardening the container through applying our SECCOMP profiles 
can mitigate previously disclosed Linux kernel CVEs. This can serve as a sign 
as how effective our approach can be in limiting the attacker's capabilities.

You can use the filterToCveProfile.py script to map the generated SECCOMP profile 
to the mitigated CVEs.

```
python3.7 filterProfileToCve.py -c cve.files/cveToStartNodes.csv.validated -f profile.report.details.csv -o results -v cve.files/cveToFile.json.type.csv --manualcvefile cve.files/cve.to.syscall.manual --manualtypefile cve.files/cve.to.vulntype.manual -d
```

-c: Path to the file containing a map between each CVE and all the starting 
nodes which can reach the vulnerable point in the kernel call graph

-f: This file is generated after you run Confine for a set of containers. 
It can be found in the results path in the root of the repository.

-o: Name of the prefix of the file you would like to store the results in.

-v: A CSV file containing the mapping of CVEs to their vulnerability type.

--manualcvefile: Some CVEs have been gathered manually which can be specified 
using this option.

--manualtypefile: A file containing the mapping of CVEs identified manually to 
their respective vulnerability type.

-d: Enable/disable debug mode which prints much more log messages.

***Note: The scripts required to generate the mapping between the kernel functions 
and their CVEs are in a separate repository. You do not need to recreate those results.***

## Statistics Creator
After running the createProfiles.py script we can generate different statistics 
by running the createStats script.
-r: Path to the main summarized results
-e: Path to the detailed results
-i: Path to the image list used (to extract extra information which might not exist in results)
-o: path to store output
-c: file specifying system call for each cve
-b: path to where binaries and libraries for all containers exist (this should be the same as -o in createProfiles.py)
```
python3.7 createStats.py -r results/profile.report.csv -e results/profile.report.details.csv -i images.list -o stats/ -c cveToSyscall.csv -b output/
```

## Syscall Extractor (Deprecated)
The extractSysCalls python script is the main function which takes the name of 
the required libc functions and the libc mapping file to create a seccomp 
profile.

```
python3.7 extractSysCalls.py -f nginx.container.nginxbashwlibs -c callgraph.out
```
The required functions should be in the format of one function per line.
The callgraph file should be in the format of one function call per line. e.g. 
a->b.

## Bash Scripts
There are a couple of bash scripts used to extract information and binaries 
from the container. The container should be running for these scripts to work.

### copyAllBins.sh
This script can be used to copy binaries from inside the container to the host. 
This script takes two arguments. The first argument a file which has the binary 
paths which should be copied from inside the container. The second argument 
should be the path to store the extracted files.

### copyAllBinsWithLibs.sh
This script can be used to copy binaries along with their dependent libraries 
from the container to the host. It also takes two arguments. The first argument 
a file which has the binary paths which should be copied from inside the 
container.

### extractAllImportedFuncs.sh
This script can be used to extract all the imported functions of ELF files. 
The folder of the ELF files can be passed as the first argument. The output 
file can be specified by the second argument.


## Paper for reference:
Please consider citing our paper if you found our tool set useful.
```
@inproceedings{confineraid20,
year={2020},
booktitle={Proceedings of the International Conference on Research in Attacks,
Intrusions, and Defenses (RAID)},
title={Confine: Automated System Call Policy Generation for Container Attack
Surface Reduction},
author={Ghavamnia, Seyedhamed and Palit, Tapti and Benameur, Azzedine and
Polychronakis, Michalis}
}
```

## Authors

* **SeyedHamed Ghavamnia**
* **Tapti Palit**

