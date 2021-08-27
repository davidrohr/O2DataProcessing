THis repository contains the PDP workflows to run on the EPN (in the future also on the FLP) and the parse script which parses the description files and creates the DDS XML files

Terminology
- A **workflow** refers to a single DPL workflow binary, or multiple workflows binaries merged with the `|` syntax, or a shell script starting such a workflow.
- A **full topology** refers to the final XML file that is passed to DDS to start up a processing chain for one single partition on the EPN.
- A **partial topology** is the XML file created by DPL with the `--dds` option.

Folder structure:
- **common** contains common scripts that can be used by all workflows, most importantly common environment variable scripts.
- **production** contains the production workflows for global runs, which are maintained by PDP experts.
- **tools** contains the parse script and auxiliary tools.
- **testing** contains scripts for tests / standalone runs maintained by detectors or privately.

Topology descriptions and description library files:

Remarks:
- The repository does not store full topologies, but they are created on the fly. Users can cache the resulting full topology XML files.
- The defaults (particularly also those set in the common environment files in the `common` folder) are tuned for running on a laptop / desktop.
- Workflows support 3 run modes selected via the `WORKFLOWMODE` env variable:
  - **run** (default): run the workflow
  - **print**: print the final workflow command to the console
  - **dds**: create a partial topology.

Configuring / selecting workflow in AliECS:
There are 3 ways foreseenm to configure the workflow in AliECS: (currently only the manual XML option exists)
- **hash of workflow repository**: In this mode, the following settings are configured in AliECS, and they uniquely identify a workflow:
  - A **commit hash** to the workflow repository (this can also be a tag, and in the case of production workflows it is required to be a tag).
  - The path of a **description library file** (relative path inside the workflow repository).
  - The **workflow name** inside the description library file.
  - A **detector list**: comma-separated list of detectors participating in the run, defaulting to `ALL` for all detectors.
  - **workflow parameters**: text field passed to workflow as environment variable for additional options.
    
As another abstraction layer above the workflows we will have topology descriptions, which we will house in the same repository. A parser, together with the `–dds` option of DPL and the `odc-topo` tool will be able to create the full topology XML file from such a description. We will store description library files in the repository, which can contain multiple topology descriptions identified by a name. In AliECS we will select the full topology to run by the tripple consisting of:
        commit hash in the workflow repository
        absolute path to a description library file in the repository
        name of the topology description in the file
    The topology description will consist of
        The O2 version to use (both for creating the XML files with the `–dds` option and for running O2 itself), there should be an alias `latest` for the latest O2 version available.
        A list of workflows, in the form of commands to run to create XML files by the `–dds` option. The commands shall be relative paths inside the workflow repository relative to topology library file. The env options used to configure the workflow are prepended in normal shell syntax.
        Each workflow is amended with the following parameters:
            Zone where to run the workflow (calib / reco)
            For reco:
                Number of nodes to run this workflow on
                Minimum number of nodes required forthe workflow (in case of node failure)
            For calib:
                Number of physical cores to be reserved on the node to run the workflow.
                Name of the calibration (to be used for identifying on which node a certain calibration is running)
            Additional option where to run this workflow (see Q2)
    The synchronous processing workflow will be the same as used for the full system test. An identical copy of the script will be in the O2 repo for the full system test and in the workflow repo for production. Changes should be synchronized manually.
    ODC/DDS should allocate as many nodes as necessary to have sufficient CPU cores for the calibration workflows. The different calibration workflows may or may not run on the same node.
    All reco workflows in the description should request the same number of EPN processing nodes. The minimum number may vary, and in that case the largest minimum is the limit. ODC/DDS should allocate as many nodes as indicated, and start all workflows and the TfBuilder on all of the nodes. If due to node failures the number of operating nodes is less than the (largest) minimum, the run should stop.
    We need an option to have open parameters in the workflow, which are filled when the workflow is started (e.g. address of the calibration nodes).
    The nodes in the reco-zone should  be numbered, and the number and count of the nodes must be passed to all the dpl-workflows being started. The simplest way is probably by an open parameter in the XML which is replaced by node number/count.

An example for the topology library file could look like:
topologies.txt
```
demo-full-topology: o2-v1.0.1 "SHMSIZE=320000000000 full-system-test/dpl-workflow.sh" reco-zone 128 126; "SHMSIZE=2000000000 calibration/some-calib.sh" calib-zone 20c; "SHMSIZE=2000000000 calibration/other-calib.sh" calib-zone 10c;
other-topology: o2-v1.0.2 "tpc-test/tpc-standalone-test-1.sh" reco-zone 2 2
```
AliECS-config:
```
commit=xxxx file=topologies.txt topology=demo-full-topology
```

Still to be decided:

    Exact format of the workflow description. (XML or JSON would be an option, but I believe that it is actually so simple that a text syntax as above is sufficient.
    How to parse the description. Will this be done by odc-topo, or do we have a python/bash parser that runs odc-topo at the end?
    How do we pass to the reco workflow which calibration workflow runs on which node? This is required such that the workflows know where to send the data to. Ideally we would have some placeholders in the created DDS XML files, which can then be replaced dynamically after the dynamic node allocation in the EPN. (Something similar might be needed for QC).
    How to pass node number / count to the DPL workflow.
