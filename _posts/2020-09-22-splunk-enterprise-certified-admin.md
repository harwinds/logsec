---
layout: post
title: "Splunk Enterprise Certified Admin Notes"
comments: true
excerpt_separator: <!--more-->
---
I passed my Splunk Enterprise Certified Admin `SPLK 1003` cert on September 11th, 2020. While I have been actively working with Splunk as part of my job,
clearing the exam however requires a consistent and well planned effort. 

All these points are taken from the exam blueprint: <https://www.splunk.com/pdfs/training/Splunk-Test-Blueprint-Admin-v.1.1.pdf>{:target="_blank"} updated with either with Splunk documentation link I studied to prepare for that specific part or youtube video I found useful. Along with these I also bought Udemy Course 
[The Complete Splunk Enterprise Certified Admin Course 2020](https://www.udemy.com/share/102vGYBkUZeFxVRnQ=/){:target="_blank"}

<!--more-->
From the `Splunk Enterprise Certified Admin` documentation: <https://www.splunk.com/en_us/training/certification-track/splunk-enterprise-certified-admin/overview.html>{:target="_blank"}, a Splunk Enterprise Certified Admin manages various components of Splunk Enterprise on a daily basis, including license management, indexers and search heads, configuration, monitoring, and getting data into Splunk. This certification demonstrates an individual's ability to support the day-to-day administration and health of a Splunk Enterprise environment.

### 1.0 Splunk Admin Basics (5%)
* 
{:toc}

1. Identify Splunk components
Components of Splunk Enterprise: <https://docs.splunk.com/Documentation/Splunk/8.0.5/Capacity/ComponentsofaSplunkEnterprisedeployment>{:target="_blank"}
YouTube: <https://youtu.be/a-_LHJU07VU>{:target="_blank"}

| Processing Layer Components   |
|-------------------------------|
| Indexer                       |
| Search Head                   |
| Universal Forwarders (Agents) |
| Heavy Forwarders              | 

> Indexer
*	Indexers index and store data
*	In a distributed environment: 
    * Resides on dedicated machines
    * Can be clustered or independent
    * Clustered indexers are known as peer nodes

> Search Head
*  Search head manage search requests from users
*  Distribute searches across indexers
*  Consolidates the results from the indexers

Universal Forwarders (Agents)

> Heavy Forwarders
*  Forwarders forward data from one Splunk component to another
*  From a source system to an indexer or indexer cluster
*  From a source system directly to a search head (SH)

| Management Layer Components   |
|-------------------------------|
| License Manager               |
| Deployment Server             |
| Cluster Master                |
| Heavy Forwarders              | 

> License Manager
*  Clients are called license slaves
*  Manages license pools and stacks

> Deployment Server
*  Centralized configuration manager
*  Manages deployments apps for clients
*  Configured through the forwarder management interface

> Cluster Master:
*  Manage index clusters
*  Coordinates the activities within the cluster
*  Manages data replication
*  Manges buckets (storage) for the cluster
*  Handles updates for the indexer cluster

> *Every Splunk component is built using Splunk Enterprise. It’s only a matter of configuration
> Except the Universal Forwarder, which is a specialized light Splunk Enterprise installation.*

### 2.0 License Management (5%)
1. Identify license types <https://docs.splunk.com/Documentation/Splunk/8.0.5/Admin/TypesofSplunklicenses>{:target="_blank"}
2. Understand license violations <https://docs.splunk.com/Documentation/Splunk/8.0.5/Admin/Aboutlicenseviolations>{:target="_blank"}

| License Types |
|---|
| The Splunk Enterprise license |
| The Splunk Enterprise infrastructure license |
| The Splunk Enterprise Trial license  |
| Sales Trial license  |
| Dev/Test licenses  |
| Free license  |
| Forwarder license |

### 3.0 Splunk Configuration Files (5%)
1. Describe Splunk configuration directory structure
2. Understand configuration layering
3. Understand configuration precedence <https://docs.splunk.com/Documentation/Splunk/8.0.5/Admin/Configurationfiledirectories>{:target="_blank"}
4. Use btool to examine configuration settings <https://docs.splunk.com/Documentation/Splunk/8.0.5/Troubleshooting/Usebtooltotroubleshootconfigurations>{:target="_blank"}

**Precedence within global context: When the file context is global, directory priority descends in this order:**

| Global Context                             |
|--------------------------------------------|
| System local directory -- highest priority |
| App local directories                      |
| App default directories                    |
| System default directory -- lowest priority|

Location
- System local directory location: $SPLUNK_HOME/etc/system/local
- App local directories location: $SPLUNK_HOME/etc/apps/<app_name>/local
- App default directories
- System default directory location: $SPLUNK_HOME/etc/system/default

**Precedence within app or user context: For files with an app/user context, directory priority descends from user to app to system:**

| App/User Context                                                                              |
|-----------------------------------------------------------------------------------------------|
| User directories for current user -- highest priority                                         |
| App directories for currently running app (local, followed by default)                        |
| App directories for all other apps (local, followed by default) -- for exported settings only |
| System directories (local, followed by default) -- lowest priority                            |

### 4.0 Splunk Indexes (10%)
1. Describe index structure
2. List types of index buckets: Hot, Warm, Cold, Frozen, Thawed <https://docs.splunk.com/Documentation/Splunk/8.0.5/Indexer/HowSplunkstoresindexes>{:target="_blank"}

3. Check index data integrity <https://docs.splunk.com/Documentation/Splunk/8.0.5/Security/Dataintegritycontrol>{:target="_blank"}

4. Describe indexes.conf options
5. Describe the fishbucket <https://www.splunk.com/en_us/blog/tips-and-tricks/what-is-this-fishbucket-thing.html>{:target="_blank"}
6. Apply a data retention policy

### 5.0 Splunk User Management (5%)
1. Describe user roles in Splunk
2. Create a custom role
3. Add Splunk users <https://docs.splunk.com/Documentation/Splunk/8.0.5/Admin/Aboutusersandroles>{:target="_blank"}

### 6.0 Splunk Authentication Management (5%)
1. Integrate Splunk with LDAP <https://docs.splunk.com/Documentation/Splunk/8.0.5/Security/SetupuserauthenticationwithLDAP>{:target="_blank"}

2. List other user authentication options
3. Describe the steps to enable Multifactor Authentication in Splunk <https://docs.splunk.com/Documentation/Splunk/8.0.5/Security/AboutMultiFactorAuth>{:target="_blank"}

### 7.0 Getting Data In (5%)
1. Describe the basic settings for an input: <https://docs.splunk.com/Documentation/Splunk/8.0.5/Data/Modifyinputsettings>{:target="_blank"}
2. List Splunk forwarder types <https://docs.splunk.com/Documentation/Splunk/8.0.5/Forwarding/Typesofforwarders>{:target="_blank"}
- Universal
- Heavy
- Light
	
3. Configure the forwarder <https://docs.splunk.com/Documentation/Forwarder/8.0.5/Forwarder/Configuretheuniversalforwarder>{:target="_blank"}
4. Add an input to UF using CLI

### 8.0 Distributed Search (10%)
1. Describe how distributed search works
2. Explain the roles of the search head and search peers
With distributed search, a Splunk Enterprise instance called a search head sends search requests to a group of indexers, or search peers, which perform the actual searches on their indexes. The search head then merges the results back to the user.

3. Configure a distributed search group <https://docs.splunk.com/Documentation/Splunk/8.0.5/DistSearch/Distributedsearchgroups>{:target="_blank"}
4. List search head scaling options

### 9.0 Getting Data In – Staging (5%)
1. List the three phases of the Splunk Indexing process
<https://docs.splunk.com/Documentation/Splunk/8.0.5/Indexer/Howindexingworks>{:target="_blank"}
2. List Splunk input options

### 10.0 Configuring Forwarders (5%)
1. Configure Forwarders
<https://youtu.be/S4ekkH5mv3E>{:target="_blank"}
2. Identify additional Forwarder options

### 11.0 Forwarder Management (10%)
1. Explain the use of Deployment Management
2. Describe Splunk Deployment Server
3. Manage forwarders using deployment apps
4. Configure deployment clients
<https://docs.splunk.com/Documentation/Splunk/8.0.5/Updating/Configuredeploymentclients>{:target="_blank"}
5. Configure client groups
6. Monitor forwarder management activities

### 12.0 Monitor Inputs (5%)
1. Create file and directory monitor inputs
<https://docs.splunk.com/Documentation/Splunk/8.0.5/Data/MonitorfilesanddirectorieswithSplunkWeb>{:target="_blank"}
2. Use optional settings for monitor inputs
3. Deploy a remote monitor input

### 13.0 Network and Scripted Inputs (5%)
1. Create network (TCP and UDP) inputs: <https://docs.splunk.com/Documentation/SplunkCloud/8.0.2007/Data/Monitornetworkports>{:target="_blank"}
2. Describe optional settings for network inputs
3. Create a basic scripted input

### 14.0 Agentless Inputs (5%)
1. Identify Windows input types and uses
2. Describe HTTP Event Collector

### 15.0 Fine Tuning Inputs (5%)
1.  Understand the default processing that occurs during input phase <https://docs.splunk.com/Documentation/Splunk/8.0.6/Deploy/Datapipeline>{:target="_blank"}
2. Configure input phase options, such as sourcetype fine-tuning and character set
Encoding

### 16.0 Parsing Phase and Data (5%)
1. Understand the default processing that occurs during parsing
2. Optimize and configure event line breaking
3. Explain how timestamps and time zones are extracted or assigned to events
4. Use Data Preview to validate event creation during the parsing phase

### 17.0 Manipulating Raw Data (5%)
1. Explain how data transformations are defined and invoked 
2. Use transformations with props.conf and transforms.conf to: 
- Mask or delete raw data as it is being indexed 
- Override sourcetype or host based upon event values 
- Route events to specific indexes based on event content 
- Prevent unwanted events from being index
3. Use SEDCMD to modify raw data
