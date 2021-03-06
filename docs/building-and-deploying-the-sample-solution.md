# Building and Running the IoT Sample Solution

This document describes how to build and run the sample IoT solution. The solution simulates a large number of devices sending environmental readings for apartments in a smart building. The IoT system implements a number of data flows:

- Capturing the raw event data and quickly saving it to storage, 

- Implementing a *warm path* that performs some intermediate processing and summarizes the raw event data (this processing will take a finite time and the results might not be immediately available),

- Implementing a *cold path* where the data is stored for further ad-hoc analysis. 

Click [here](./overview-of-architecture-and-components.md) for more information about these flows.  

The simulator enables you to test the following scenarios:

1. All devices sending readings that are within the expected range. These readings simply need to be recorded so that they can be displayed on a dashboard or used for further analysis.

2. Some devices send readings indicating that the temperature is excessive (over 30 degrees Celsius). As well as being recorded, these readings may prompt some more immediate action.

You can download the code for the solution [here](../src).

You should also download the [provisioning scripts](../provision) that create the Azure assets required by the solution.

The solution comprises the following projects:

- ScenarioSimulator. This project contains the logic that implements each of the scenarios. You can configure the parameters used by the simulator (number of devices, simulation duration, warm up time, and so on) by editing the mysettings.config file in this project.

- Devices.Events. This project defines the events that the simulated devices for each of the scenarios can raise.

- ScenarioSimulator.ConsoleHost (in the RunFromConsole folder). This project provides the user interface to the simulator. It implements a menu that enables the user to run each of the scenarios described above.

- ColdStorage.ConsoleHost (in the RunFromConsole folder) and ColdStorage. These projects implement a cold storage processor that captures event data directly from Event Hub and sends it to blob storage for later analysis.

- Core. This project implements generic, reusable logic that is referenced by the other projects in the solution.

- ScenarioSimulator.Tests, ColdStorage.Tests and Core.Tests (in the UnitTests folder). These projects contain unit tests to verify that the simulator and cold storage processor are configured and running correctly.

## Prerequisites

To use the sample solution, you must have an Azure account with a subscription. You should also have installed the following software on your computer:

- [Visual Studio](https://www.visualstudio.com/)

- [Azure SDK for .NET (Visual Studio tools)](http://azure.microsoft.com/downloads/)

- [Azure Powershell](https://azure.microsoft.com/documentation/articles/powershell-install-configure/)

## Building the Sample Solution

To build and deploy the system, perform the following steps:

1. Download and install the NuGet packages and assemblies required by the solution.

2. Create the required Azure assets and services.

3. Verify the Stream Analytics configuration.

4. Verify the Azure SQL Database configuration.

5. Verify the HDInsight configuration.

6. Create the configuration file with your Azure account information and build the solution.

The following sections describe these steps in more detail.

### Downloading and Installing the Required Assemblies

- Using Visual Studio, open the IoTJourney solution.

- Rebuild the solution. This action downloads the various NuGet packages referenced by the solution and installs the appropriate assemblies.

***THE MESSAGE HIGHLIGHTED BELOW WILL BE CHANGED TO SOMETHING MORE MEANINGFUL***
> **Note:** The rebuild process should report an error: *Before building this project, please copy mysettings-template.config to mysettings.config ...* You will do this in a later step. If the rebuild process displays any other errors then clean the solution and rebuild it again.

### Creating the Required Azure Assets and Services

- Start Azure PowerShell as administrator.

> **Note:** Windows Powershell must be configured to run scripts. You can do this by running the following PowerShell command before continuing:
> 
> `set-executionpolicy unrestricted`

- Move to the folder containing the provisioning scripts.

- Run the following command:

	`.\Provision-All.ps1`

-  At the *SubscriptionName* prompt, enter the name of the subscription that you are using with your Azure account.

- At the *ServiceBusNamespace* prompt, enter the name of the Service Bus namespace you wish to create to hold the event hub.
 
- At the *StorageAccountName* prompt, enter the name of the storage account you wish to create to hold the data generated by the simulator.

- At the *SqlServerName* prompt, enter the name of the Azure SQL Database server you wish to create to hold the data for warm processing.

- At the *SqlDatabasePassword* prompt, enter a new password for the Azure SQL Database server.

- At the *HDInsightClusterName* prompt, enter the name of the HDInsight Cluster you wish to create to process event data.

- In the Sign in to Windows Azure Powershell dialog box, provide the credentials for your Azure account. 

- Wait while the script creates the Azure resources.

- In the Windows PowerShell credential request dialog box, enter a new password for the admin account for the HDInsight cluster to be created.

> **Note:** The password must be at least 10 characters long, contain a mixture up upper and lower case letters and numbers, and include at least one special character. If you fail to provide a sufficiently complex password, the provisioning script will fail to create the cluster and instead report a *PreClusterCreationValidationFailure*.

- Wait while the HDInsight cluster is provisioned. This can take several minutes.

### Verifying the Azure Storage Configuration

The provisioning script generates Azure blob storage to hold the raw event store data after it has been captured by Stream Analytics. This data can be subjected to further analysis separately. A second Stream Analytics job utilizes reference data held in blob storage used for generating summary information. Perform the following steps to verify the configuration Azure storage:

- Using the Visual Studio Server Explorer window, connect to your Azure account.

- Expand the Storage node, expand the node that corresponds to your storage account, expand Blobs, and verify that the following containers have been created (you may need to refresh the display if the containers do not appear).

	- container01
	
	- container01refdata

	- iot-hdicontainer01

	- *HDIInsightClusterName* (the value that you specified for the HDI cluster name when you ran the provisioning script).

- Double-click container01refdata to display the contents pane.

***TBD - THE PROVISIONING SCRIPT NEEDS CREATE THE FOLDER AND FILE SHOWN BY THESE NEXT TWO STEPS***
- On the contents pane, verify that the container has a folder named fabrikam.

- Double-click the fabrikam folder and verify that it contains a file named buildingdevice.json. This file contains the reference data used by the summarizing element of the solution.

### Verifying the Azure SQL Database Configuration

The provisioning script creates an Azure SQL Database to hold the rolling summary data of the average temperature in each building. Perform the following steps to verify the configuration of the database and server:

- Using the Azure web portal, open the SQL Databases page.

- Verify that a database names fabrikamdb01 has been successfully created and that it is located in the server that you specified in response to the *sqlServerName* prompt when you ran the provisioning script.

- Select the fabrikamdb01 database, and then in the command bar click Manage. The Azure SQL Database Management portal should open.

- On the log on page, in the Username box type fabrikamdbuser01, in the Password box enter the password that you specified in response to the *SqlDatabasePassword* prompt when you ran the provisioning script, and then click Log On.

- In the left-hand pane, click Design. In the main pane, verify that a table named BuildingTemperature has been created (the table should be empty; the row count property should be 0).

- Click Edit to display the table schema. The table should contain the following columns:

	- Id (integer)

	- BuildingId (varchar(100))

	- LastObservedTime (datetime)

	- Temperature (decimal(3,1))

### Verifying the Stream Analytics Configuration

The provisioning script should configure Stream Analytics automatically to receive information from two input sources; event data from the event hub and JSON formatted reference data held in the container01refdata container in blob storage (the JSON data is used by the summary job in Stream Analytics to provide lookups that can make the raw event data more understandable). The stream analytics job writes records of each event to blob storage using JSON format, and a summary showing the average temperature observed by all devices in each building during a 5 minute time window to the BuildingTemperature table in the fabrikamdb01 Azure SQL Database.

To verify that the configuration was successful, perform the following steps:

- Using the Azure web portal, open the Stream Analytics page.

***TBD - THE JOB WILL BE SPLIT INTO TWO PIECES***
- On the Stream Analytics page, verify that a Stream Analytics job called fabrikamstreamjob01 has been created.

- Click the job, and then in the menu bar click Configure

- On the Configuration page, verify that the storage account is set to the value that you specified when you ran the provisioning script (the name that you entered in response to the *StorageAccountName* prompt).

- In the menu bar click Inputs.

- On the Inputs page, verify that two inputs named input01 and input02 have been created. The type of the input should be Event Hub and the type of input02 should be Blob storage.

- Click input01. 

- On the input01 page, in the General section, verify that:

	-  The service bus namespace is set to the value that you specified in response to the *ServiceBusNamespace* prompt when you ran the provisioning script.

	-  The event hub name is set to eventhub01.

	-  The event hub policy name is set to ManagePolicy.

	-  The event hub consumer group is set to consumergroup01.

- In the Serialization section, verify that the event serialization format is set to JSON.

- Return to the job page, and then click input02. 

- On the input02 page, in the General section, verify that:

	-  The storage account Bus namespace is set to the value that you specified in response to the *StorageAccountName* prompt when you ran the provisioning script.

	-  The container is set to container01refdata.

	-  The path pattern is set to fabrikam/buildingdevice.json.

- In the Serialization section, verify that the event serialization format is set to JSON.

- Return to the job page, and then in the menu bar click Query

- On the Query page, verify that the following queries are defined:
	```
	SELECT * INTO output01 FROM input01 TIMESTAMP BY TimeObserved;

	SELECT AVG(I1.Temperature) as Temperature, Max(I1.TimeObserved) as LastObservedTime, I2.BuildingId 
	INTO output02 
	FROM input01 I1 TIMESTAMP BY TimeObserved
	JOIN input02 I2 On I1.DeviceId = I2.DeviceId
	GROUP BY TumblingWindow(s,5), I2.BuildingId
	```

- In the menu bar click Outputs.

- On the Outputs page, verify that the following two outputs are defined:

	- output01 with an output type of Blob Storage

	- output02 with an output type of SQL Database

- Click output01. On the output01 page, in the General section, verify that:

	-  The storage account is set to the value that you specified in response to the *StorageAccountName* prompt when you ran the provisioning script.

	-  The container is set to container01.

	-  The filename prefix is set to fabrikam.

	- In the Serialization section, verify that the event serialization format is set to JSON.

- Return to the Outputs page.

- Click output02. On the output02 page verify that:

	- The server name is set to the value that you specified in response to the *sqlServerName* prompt when you ran the provisioning script.

	- The database name is set to fabrikamdb01.

	- The username is set to fabrikamdbuser01.
	
	- The tablename is set to BuildingTemperature.

### Verifying the HDInsight Configuration

The provisioning script creates an HDInsight cluster that uses Hadoop services running a Hive query to process event data. Perform the following steps to verify the configuration of the HDInsight cluster:

- Using the Azure web portal, open the HDInsight page.

- Verify that the HDInsight cluster has been successfully created.

- Click the cluster. 

- In the menu bar click Dashboard.

- On the Dashboard page, verify that the cluster is linked to the storage account that you specified in response to the *StorageAccountName* prompt when you ran the provisioning script.

- In the menu bar, click Configuration.

- On the Configuration page, verify that Hadoop Services are enabled (On)

- In the menu bar, click Scale.

- On the Scale page, verify that the instance count is set to 2.

### Updating the Configuration File for the Unit Tests

- In Visual Studio, open the *app.config* file in the ColdStorage.Tests project in the UnitTests folder. 

- In the <appSettings> section, in the value for the storageconnectionstring key, replace the text *[YourStorageAccountName]* with the name of the storage account that you specified when you ran the provisioning script.

- Using the Azure portal, find the primary account key for the storage account that was created by the provisioning script.

- In Visual Studio, replace the text *[YourStorageAccountKey]* with the primary account key.

- Save the configuration file and rebuild the solution. It should now build successfully.

## Testing and Running the Solution

The following sections describe how to test and run the simulator and cold storage processor.

### Running the Unit Tests

To run the unit tests, on the Visual Studio menu bar click Test, click Run, and then click All Tests. All tests should succeed without errors. If any tests fails, verify that you have set the configuration parameters in mysettings.config correctly.

### Running the Simulator to Generate Events

Perform the following steps to run the simulator:

- Using the Azure web portal, open the Stream Analytics page.

- Select fabrikamstreamjob01, and on the command bar click Start.

- In the Start Output dialog box, select Job Start Time and then click the tick button.

- Wait for the Stream Analytics job to start before continuing.

- Using Visual Studio, run the ScenarioSimualtor.ConsoleHost project.

- On the menu, select option 1 (*Run NoErrorsExpected*). The simulator will generate events and echo their contents to the screen. The simulator should not report any exceptions.

- Allow the simulator to run for a few minutes and then press q.

- Press Enter to return to the menu.

- Leave the simulator running.

### Verifying the Raw Event Data Output by Stream Analytics

Perform the following steps to verify that events are being processed correctly by Stream Analytics:

- Using the Visual Studio Server Explorer window, connect to your Azure account.

- Expand the Storage node, expand the node that corresponds to your storage account, expand Blobs, and then double-click container01 to display the container01 contents pane.

- On the container01 contents page, verify that the container has a folder named fabrikam.

- Double-click the fabrikam folder and verify that it contains one or more JSON files (files with random name but with the .json suffix).

- Double-click the most recent file to download and view the contents. Verify that the file contains a number of JSON formatted event records; there should be one record for each event that was generated when the simulator ran, although you probably didn't count them at the time! 

	This data was generated by the first Stream Analytics query:

	```
	SELECT * INTO output01 FROM input01 TIMESTAMP BY TimeObserved
	```

	This query simply copies the event data received from the devices to blob storage.

> **Note:** The data for each event comprises the ID of the device that reported the temperature, the value of the temperature, the date and time at which the data was processed, the event hub partition used to process the data, and the date and time at which the data was received by the event hub.

### Analyzing the Raw Event Data by Using a Hive Query

Perform the following steps to analyze the raw event data by using a Hive query:

- Start Azure PowerShell as administrator.

> **Note:** Windows Powershell must be configured to run scripts. You can do this by running the following PowerShell command before continuing:
> 
> `set-executionpolicy unrestricted`

- Move to the folder containing the source code for the solution scripts, and then move to the Validation\HDInsight subfolder.

- Run the following command:

	`.\hivequeryforstreamanalytics.ps1`

- At the *subscriptionName* prompt, enter the name of the subscription that  owns the storage account holding the blob data used by the simulator.

- At the *storageAccountName* prompt, enter the name of the storage account.

- At the *containerName* prompt, type container01 (this is the name of the container that the simulator uses to hold the blob data).

-  At the *clusterName* prompt, enter the name of the Hadoop cluster.

-  In the Sign in to Windows Azure Powershell dialog box, provide the credentials for your Azure account.

- Verify that the message Successfully connected to cluster *cluster name* appears (where *cluster name* is the name of your Hadoop cluster), and then wait for the Hive query to complete. The results should appear on the standard output consisting of a series of pairs; the ID of a device and the number of events that the device generated.

### Analyzing the Summary Data by Using a SQL Query

Perform the following steps to examine the summary data generated by Stream Analytics:

- Using the Azure web portal, open the SQL Databases page.

- Select the fabrikamdb01 database, and then in the command bar click Manage. The Azure SQL Database Management portal should open.

- On the log on page, in the Username box type fabrikamdbuser01, in the Password box enter the password that you specified in response to the *SqlDatabasePassword* prompt when you ran the provisioning script, and then click Log On.

- In the menu bar, click New Query.

- In the query editor, type `SELECT * FROM dbo.BuildingTemperature`

- In the menu bar, click Run. The results pane should display one or more rows displaying the average recorded temperature across all devices in each building over the last 5 seconds.

	This data was generated by the second Stream Analytics query:

 	```
	SELECT AVG(I1.Temperature) as Temperature, Max(I1.TimeObserved) as LastObservedTime, I2.BuildingId 
	INTO output02 
	FROM input01 I1 TIMESTAMP BY TimeObserved
	JOIN input02 I2 On I1.DeviceId = I2.DeviceId
	GROUP BY TumblingWindow(s,5), I2.BuildingI
	```

	This query calculates the average temperature returned by each device but groups the devices by the buildings in which they are located. The reference data in buildingdevice.json file in blob storage contains a list of devices in each building and provides the mapping between devices and buildings used by the `JOIN` clause. 

> **Note:** If you used the default number of devices (100) in the mysettings.config file, you will only see data for Building 0. To see data for more buildings, increase the number of devices  by 100 for each additional building.

### Running the Simulator to Generate Events Indicating High Temperature

- Return to the Simulator (or restart it if you closed it earlier).

- On the menu, select option 2 (*Run ThirtyDegreeReadings*). The simulator will generate events that indicate that the temperatures reported by devices are 30 degrees Celsius (too hot to be comfortable).

- Allow the simulator to run for a few minutes and then press q.

- Press Enter to return to the menu.

- On the menu, select option 3 to quit the simulator.

- Return to the Azure SQL Database Management portal displaying the query `SELECT * FROM dbo.BuildingTemperature`, and in the menu bar click Run.

- Verify that the temperatures now reported by each building are 30 degrees Celsius.

### Running the Cold Storage Processor

Perform the following steps to run the cold storage processor. 

1. Using Visual Studio, run the ColdProcessor.ConsoleHost project.

- On the menu, select option 1 (*Provision Resources*). The cold storage processor will check that the resources it requires are available, or create them if necessary.

- Press Enter to return to the menu.

- On the menu, select option 2 (*Run Cold Storage Consumer*).

- Allow the simulator to run for a few minutes and then press q.

- Press Enter to return to the menu.

- On the menu, select option 3 to quit the cold storage processor.

### Analyzing the Cold Path Data by Using a Hive Query

Perform the following steps to analyze the cold path data by using a Hive query:

- Start Azure PowerShell as administrator.

> **Note:** Windows Powershell must be configured to run scripts. You can do this by running the following PowerShell command before continuing:
> 
> `set-executionpolicy unrestricted`

- Move to the folder containing the source code for the solution scripts, and then move to the Validation\HDInsight subfolder.

- Run the following command:

	`.\hivequeryforcoldstorageeventprocessor.ps1`

- At the *subscriptionName* prompt, enter the name of the subscription that  owns the storage account holding the blob data used by the simulator.

- At the *storageAccountName* prompt, enter the name of the storage account.

- At the *containerName* prompt, type coldstorage (this is the name of the container that the cold storage processor uses to save the blob data).

-  At the *clusterName* prompt, enter the name of the Hadoop cluster.

-  In the Sign in to Windows Azure Powershell dialog box, provide the credentials for your Azure account.

- Verify that the message Successfully connected to cluster *cluster name* appears (where *cluster name* is the name of your Hadoop cluster), and then wait for the Hive query to complete. The results should appear on the standard output consisting of a series of device IDs and the number of events that each device generated.



