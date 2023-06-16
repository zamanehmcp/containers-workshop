# Module 2 - Lab 1: Assessing the App and Environment for Containers

## Time: 45 minutes

Kubernetes is a popular, cloud-native container orchestration system. Adoption of Kubernetes in production environments has rapidly increased over the last several years because it helps organizations with the following:

- Streamline their operations with a single set of cloud tooling and expertise
- Achieve their DevOps, DevSecOps, and CI/CD goals
- Provides a fault-tolerant, highly-scalable platform to run mission critical applications built with almost all of the popular development languages and tools

However, cloud-native software architectures are different from traditional architectures in a variety of ways. As a result, migration of a system into a cloud-native environment is not as simple as a rehosting migration. Some applications require a great deal of effort to migrate. Establishing the goals of the migration and gathering relevant data provide you with design trade-offs to consider before modifying the application or starting the migration process. The series of questions presented below helps to gather information about the application relevant to migration. The information gathered then makes the migration straightforward by determining how to represent the application in Kubernetes.

In this exercise, you will evaluate BioGuide and the environment in which BioGuide currently operates to determine your path to containerizing the workload and deploying it to your existing Kubernetes environment. Using the guide below, complete the worksheet found [here](https://github.com/loublick-ms/containers-workshop/blob/main/Labs/Module%202%20-%20Lab%201%20Worksheet%20Assessing%20the%20Application%20and%20Environment%20for%20Containers.docx).

### Understand the Goal

A full understanding of the goals of the migration is absolutely key to a success. First, determine whether the application should be modified to support the behavior of the environment, or whether the environment should be modified to support the behavior of the application. Modifying an application that is not currently cloud native to take advantage of the benefits of cloud-native design may require extensive rework. However, it is often possible to avoid this rework, at least at first, and work around the problems with a design that is not cloud-aware.

### Some questions that should be asked before starting the migration

1. Why is the application being migrated?
2. Can the behavior of the application be modified? If not, can the cloud environment be modified to accommodate the application?

### Gather Information about the Application

The next step is to understand the scope of what needs to be migrated. What components make up the application being migrated? Is it a single service or a collection of services that work together?

After the scope is determined, the remainder of the information gathering is to gain a complete understanding of the interfaces of the application's components. The following steps should be completed for each component of the application independently.

#### Determine Network Interactions

The network is by far the most common interface in modern applications. Consider both the outbound connections that the application initiates and the inbound connections that are initiated by other applications. These interfaces must be preserved during migration so that application can function properly and users and other related applications and services are not impacted by the migration.

1. What must the service connect to in order to function?

Make sure to record the URL or IP address, the port, and the location of the resource within or outside the cluster. See the example below for how this information should be recorded, along with some example data.

| Description         | URL or IP Address  | Port  | Internal or External |
|   :---              | :---               | :---: |    :---:             |
| Azure Blob Storage  | myblob.example.com | 1234  | External             |
| Logging Service     | logger.example.com | 8080  | Internal             |

2. What services or users need access to the service?

Make sure to record the protocol, port and location of the remote user/service inside the cluster, outside the cluster, or both. See the table below for an example of how this information should be recorded, along with some example data.

| Description      | Protocol | Port  | Internal or External |
|   :---           | :---:    | :---: |    :---:             |
| Azure Sentinal   | https    | 443   | Internal             |
| Users            | https    | 443   | External             |

#### Determine Filesystem Interactions

Modern applications use filesystems for storage of configuration, static data, and dynamic data. Historically, filesystems were also used as a mechanism for communication between applications and components. Although this practice has largely been replaced by network-based interactions, it is still present in some domains and in legacy applications. Therefore, any filesystem interactions should be noted because they are very important to successful migration of these application.

1. What files and directories are read by the service?

For each file or directory, determine whether the content is static, configuration, or dynamic (i.e., modified at runtime). See the table below for an example of how this information should be recorded, along with some example data.

| Directory/File           | Static/Config/Dynamic |
|   :---                   | :---:                 |
| /etc/myapp/config.yaml   | Config                |
| /wwwroot/myapp/views/css | Static                |

2. What files are modified by the service?

Make sure to record whether the modifications must survive service restarts (i.e., be persistent) or if they can be transient. Also record whether the changes must bevisible to other applications or executables, or only be visible to the service instance itself. See the table below for an example of how this information should be recorded, along with some example data.

| Directory/File           | Transient/Persistent | Visibility  |
|   :---                   | :---:                |   :---:     |
| /tmp/myapp               | Transient            | Self        |
| /app-container/my-queue  | Persistent           | Self/Others |

#### Use the Data to Plan the Migration

Gathering the data in the previous section requires a deep understanding of the application being migrated. This deep understanding of the application should now be used to reflect upon the goals of the migration and reaffirm the desire to migrate. For some applications it is possible that a migration will not provide significant value without extensive refactoring of the application.

If proceeding with the migration is still deemed valuable, the actual migration involves two steps:

1. Containerization of the processes that make up the application.
2. Selecting the Kubernetes objects that will make up the components of the application in the new environment.

Step 1 will be covered later in this workshop. To proceed with step 2, here are some questions to help determine which Kubernetes objects are needed for migration of the application.

1. What controller should be used to manage the Pods?

Kubernetes has several built-in controllers for managing Pods, and each exhibits slightly different behavior.

**Replicaset** -- A Replicaset ensures that a specified number of Pods is running at any point in time. If you define that there should be five Pods running, the Replicaset will make sure that this happens. If there are any excess Pods, they get deleted.

**Deployment** -- A Deployment controller is used to run a Pod at a desired number of replicas. These Pods have no unique identities. The Deployment can specify the configuration over a standalone Pod and a Replicaset, such as what deployment strategy to use. For example, if you are upgrading an application from v1 to v2, you might consider one of the following approaches:

- Upgrade with zero downtime
- Upgrade sequentially one after the other
- Pause and resume upgrade process
- Rollback upgrade to the previous stable release

**DaemonSet** -- Like Deployment, except the Pod will be started once on every node.

**StatefulSet** Deployment controllers are suitable for managing stateless applications. Statefulsets, on the other hand, are useful when running workloads that require persistent storage. They keep unique identities for each Pod they manage and use the same identity when Pods need to be rescheduled.

**Job** -- A Kubernetes Job is a controller that supervises Pods for carrying out certain tasks. They are primarily used for batch processing. As soon as you submit a Job manifest file to the API server, the Pod will kick in and execute a task. When the task is completed, it will shut down by itself.

2. How will persistent storage be achieved?

If persistent storage is required for the application, then first consider the use of cloud-native storage options in your environment (e.g., File or Blob Storage in Azure). However, doing so will require changes to the application to support the use of these systems instead of the filesystem.

If it is not feasible to change the application, then a PersistentVolume is the way to go. If possible, a LocalPersistentVolume is a good last-resort option if performance of network-attached storage is not sufficient.

3. How should configuration be injected?

Configuration should be injected into the Pods via a ConfigMap. However, data such as certificates, passwords and other security keys should be injected via a Secret instead. Azure Key Vault is used to store these secrets and can be integrated with AKS so it is available to your application.

4. How should static files, such as certificates, be managed?

Static data should be compiled into the Docker image that makes up the Pods. Alternatively, it could be treated the same as persistent storage and loaded once, before deployment.

5. How should communication between components via filesystems be handled?

A PersistentVolume can usually be created with the necessary access permissions and mounted into the Pods that need to communicate. However, the PersistentVolume implementation that is used for a volume will impact whether that volume can be used by multiple readers or writers simultaneously.

The best way to solve this problem is to avoid it by restructuring the application to use network-based communication mechanisms. If restructuring the application is not possible (or not desirable) due to business constraints, then the processes that are communicating must be placed in the same Pod and communicate via an emptyDir volume.

6. How will log data be read from Pods?

Kubernetes expects all Pods to log everything worth logging to STDOUT and STDERR. If the application does not support the ability to modify logging options, then a sidecar container (e.g., fluentd or logstash) can be used to read logs from a shared volume and transfer them into a log aggregation system.

Microsoft Azure has a service called Azure Monitor, which is composed of a set of logging and monitoring services. Azure Monitor is available to all Azure services, including AKS, and can be enabled through configuration options in these services. Two of those Azure Monitor services, Application Insights and Container Insights, will perform logging and monitoring of containerized applications and AKS clusters respectively. 

7. Does the Kubernetes environment support network policy?

If the Kubernetes environment supports network policy, then the host and port information gathered should be used to create network policies to control communication between the pods in your cluster to scure your application and allow it to communicate with only those services that are required to function properly.
