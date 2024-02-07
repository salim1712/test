### Oracle Data Guard Automation

##### Overview

The script is designed to automate Oracle Data Guard configurations, including the setup of primary and standby databases, Data Guard Broker configurations, network settings, Oracle Wallet for secure connections, and various checks to ensure the environment's readiness and correctness. It leverages shell commands, Oracle utilities (SQL*Plus, RMAN, DGMGRL), and scripting best practices for automation and error handling. It includes operations such as setting up environment variables, validating configurations, and updating system files.

Here's a breakdown of its key components and functionalities:

**Date and Time Variables:**
- `dt` and `cdt` variables store the current date and time in specific formats. These are used for naming log files and other date-dependent operations.

**Script and Host Information:**
- Variables like `PRG`, `IP`, `HOSTNAME`, and others store metadata about the script itself and the host it's running on. This information is likely used for logging and identification purposes.

**Control File:**
- The script expects a control file named `param_file_setup_dg.ctl`. It checks for the existence of this file and terminates if the file is not found. This control file contains key configuration details required for the setup or management tasks the script is designed to perform.

**Extraction of Configuration Values:**
- It extracts multiple configuration values from the control file, such as database names, instance names, Oracle home directories, service names, wallet directories, and more. These values are crucial for configuring Oracle Data Guard and related database management tasks.

**Log Directory and File Management:**
- The script checks if a logs directory exists and creates one if it doesn't. It then manages a `.dat` file to ensure that the script is not executed multiple times in parallel.

**Logging Function:**
- A function `log_script_info` is defined to print out script execution metadata. This is useful for auditing and troubleshooting.

**Oracle Environment Setup:**
- The script appends to the `PATH` environment variable, exports Oracle-specific variables like `ORACLE_SID`, and sources the Oracle environment settings without prompting the user. This is essential for running Oracle database commands within the script.

**Modification of `/etc/oratab`:**
- It checks for an entry of the standby database instance in `/etc/oratab` and adds it if it's missing. This file is used by Oracle software to store information about Oracle instances on the machine, including their Oracle home directories and whether they should be started and stopped automatically by the dbstart and dbshut scripts.

**Error Handling and Cleanup:**
- The script includes several checks that, if failed, result in the script terminating with an error message. It also ensures that temporary files are cleaned up in case of errors.


1. The function `perform_ssh_check` is designed to verify password-less SSH connectivity between two hosts. Below is a detailed explanation of its components and functionality:

###### Function Definition: **perform_ssh_check()**
- **Purpose**: To check if password-less SSH connectivity exists between a `source_host` and a `target_host`.
- **Parameters**: 
- `source_host`: The hostname or IP address of the source machine initiating the SSH connection.
- `target_host`: The hostname or IP address of the target machine to which the SSH connection is made.

###### SSH Command Execution

The core of this function is an SSH command that attempts to execute a simple `echo` command on the `target_host` from the `source_host`. This is done by initiating an SSH session from the `source_host` to the `target_host` and executing the `echo` command to print a success message.

**SSH Options**:
`-o BatchMode=yes`: This option is used to prevent SSH from asking for a password. This is crucial for scripts meant to run automatically without human intervention.
`-o ConnectTimeout=5`: This sets a timeout for the SSH connection attempt to 5 seconds, enhancing the script's robustness by preventing it from hanging indefinitely if the connection cannot be established.

###### Error Handling and Output
- The script checks the exit status of the SSH command using the special variable `$?`. An exit status of `0` indicates that the command executed successfully, confirming that password-less SSH connectivity is in place.
- **Success Case**: If the SSH command succeeds, the function prints a success message confirming the verification of password-less SSH connectivity between the `source_host` and the `target_host`.
- **Failure Case**: If the SSH command fails (indicated by a non-zero exit status), the function prints an error message using the `log_script_info` function (presumably defined elsewhere in the script). It also removes a data file (`$DAT_FL`, not defined within this function but implied to be part of a larger script context) and exits with status `1`, signaling an error condition.

2. The function **create_password_file** has the primary objective of ensuring the existence and proper configuration of an Oracle password file (`orapw`) on both the primary and standby servers in an Oracle Database environment. This Oracle password file is crucial for database security, as it stores the passwords for users who are allowed to connect to the database instance as SYSDBA, SYSOPER, and other privileged roles.

###### Function Overview
- **Purpose**: To create an Oracle password file on the primary server if it doesn't already exist and then to copy this file to the standby server, ensuring both servers have synchronized password files for database authentication.
- **Environment Variable**: `ORACLE_SID` is expected to be set in the environment prior to running this function, indicating the Oracle System Identifier for the database instance.

###### Detailed Steps

**Password File Check on Primary Server**:
- The function first checks if the password file exists on the primary server using SSH and the `test` command. The path to the password file includes the Oracle home directory and the instance name (`orapw${PR_INSTNAME}`), which makes it specific to the database instance.

**Password File Creation**:
   - If the password file does not exist, the function remotely executes the `orapwd` utility via SSH on the primary server to create a new password file. The `orapwd` command includes parameters for specifying the file location, password, and the `force=y` option to overwrite any existing file without prompting.

 **Copy Password File to Standby Server**:
   - Regardless of whether the password file was just created or already existed, the function proceeds to copy it from the primary to the standby server using `scp`. This step ensures that the standby server has the same password file, which is crucial for scenarios like failovers or switchover operations where the standby may become the primary.

 **Error Handling and Logging**:
   - The function checks the exit status of the `scp` command to confirm successful copy operation. On failure, it logs an error message (presumably using a custom `log_script_info` function defined elsewhere) and removes a data file (`$DAT_FL`, indicating a temporary or control file used by the script), then exits with an error status to prevent further execution.

###### Key Considerations
- **Security**: The script assumes secure SSH and SCP configurations, as well as the presence of SSH keys for passwordless operations, which are best practices for automating tasks in secure environments.
- **Logging**: Output redirection or logging mechanisms are not explicitly shown within this function but are implied by references to logging functions and practices in other parts of the script.
- **Error Handling**: The function includes basic error handling to ensure clean exits and logging in case of issues, highlighting the importance of robustness in automation scripts.

The function `configure_listener_tns` is aimed at automating the configuration of Oracle network services for a Data Guard setup. It specifically focuses on the creation and management of listener and `tnsnames.ora` configurations across Standby, Primary, and Observer servers.
Here is a detailed breakdown of its operations:

###### Function Overview
- **Purpose:** Automates the setup of Oracle listener on a Standby server and configures the tnsnames.ora files on Primary, Standby, and Observer servers to ensure proper network communication between these components.
- **Operations:** The function performs several key tasks, including creating a new listener configuration, updating tnsnames.ora files, and managing SSH connections to apply changes remotely.

###### Detailed Explanation

**Logging Redirection:**
   - At the start, the function redirects stdout and stderr to a log file. This ensures all output produced during the function's execution is captured for troubleshooting and auditing.

**Creating Listener Configuration:**
   - It creates a new listener.ora file at the specified location (`${DR_ORACLE_HOME}/network/admin/listener.ora`). This file defines how the Oracle listener on the Standby server listens for incoming database connections, specifying the listener name, protocol, host, and port.

**Configuring tnsnames.ora on the Standby Server:**
   - The function generates a new tnsnames.ora file in the same directory, defining network service names for the Primary and Standby databases. This file is crucial for Oracle clients and database instances to know how to connect to different databases over the network.

**Updating tnsnames.ora on the Primary Server:**
   - Through SSH, the function checks if the Primary and Standby database entries exist in the Primary server's tnsnames.ora file. If an entry is missing, it appends the necessary configuration remotely using SSH. This step ensures that the Primary server can connect to the Standby server and vice versa.

**Copying tnsnames.ora to the Observer Server:**
   - It then uses SCP (Secure Copy Protocol) to copy the updated tnsnames.ora file from the Standby server to the Observer server. The Observer server, used in Oracle Data Guard configurations for monitoring and managing failover, requires this file to connect to both Primary and Standby databases.

**Restarting the Listener:**
   - Finally, the function stops and then starts the Oracle listener on the Standby server to apply the new configurations. Restarting the listener is necessary for changes to take effect.

###### Summary of Key Components

- **Listener Configuration:** Enables the Standby server to listen for Oracle database connections.
- **tnsnames.ora Configuration:** Facilitates network connectivity between Oracle database instances by defining network addresses and service names for clients and databases to use when connecting.
- **Remote Management:** Uses SSH for remote checks and updates, and SCP for file transfers, demonstrating an automated, cross-server configuration approach.

3. The functions `create_wallet` and `add_wallet` are designed for managing Oracle wallets in the context of an Oracle database environment, particularly within a configuration involving Primary, Standby, and Observer servers. Oracle wallets are secure containers used to store authentication and signing credentials, including network service names, trusted certificates, and private keys.

###### `create_wallet` Function
- **Purpose**: Checks for the existence of an Oracle wallet by looking for `ewallet.p12` or `cwallet.sso` files in the designated wallet directory. If the wallet does not exist, the function creates a new Oracle wallet in auto-login mode, which doesn't require a password for access.
- **Operations**:
    - Redirects all output to a log file for auditing.
    - Checks for existing wallet files.
    - Uses `orapki` tool to create a new wallet with the `-auto_login` flag and a specified password, ensuring the operation's output isn't printed to the console.

###### `add_wallet` Function
- **Purpose**: Ensures the `sqlnet.ora` configuration file on the Standby, Primary, and Observer servers contains the `WALLET_LOCATION` directive, pointing to the wallet's directory. It then populates the wallet with credentials for different services and copies the wallet to the Primary and Observer servers.
- **Operations**:
    - Redirects output to a log file.
    - Checks and updates `sqlnet.ora` for the presence of `WALLET_LOCATION` on all servers.
    - Uses SSH for remote operations on the Primary and Observer servers to update their `sqlnet.ora` files if necessary.
    - Utilizes the `mkstore` tool to add credentials to the wallet for different TNS (Transparent Network Substrate) entries, indicating how to connect to various database services securely without exposing passwords.
    - Copies the wallet files to the Primary and Observer servers, ensuring they have the necessary credentials for secure connections.

###### Detailed Steps in `add_wallet`
**Update `sqlnet.ora` Configuration**: Adds or verifies the existence of `WALLET_LOCATION` in the `sqlnet.ora` file, allowing the Oracle network services to locate and use the wallet for authentication.
**Add Credentials to Wallet**: Uses `mkstore -createCredential` to add database connection credentials to the wallet, facilitating password-less authentication.
**Copy Wallet to Other Servers**: Securely transfers the wallet files to the Primary and Observer servers, ensuring these systems can authenticate securely to the database without manual password entry.

###### Common Themes and Practices
- **Security and Automation**: The scripts emphasize secure handling of credentials and automated deployment, enhancing security and operational efficiency.
- **Error Handling**: Demonstrates checking operation success and provides feedback on failures, crucial for troubleshooting and reliability.
- **Logging**: Redirects output to a log file, a common practice for auditing and troubleshooting automated scripts.

4. The function `perform_pre_checks` conducts a series of pre-checks relevant to an Oracle Database environment, especially in configurations involving primary, standby, and observer servers. This function aims to ensure system readiness and compatibility before proceeding with operations such as Oracle Data Guard configuration or similar database management tasks. Below is a breakdown of the key operations performed by the function:

###### Local and Remote Hostname Checks
- Verifies the presence of the DR (Disaster Recovery or Standby) host entry in the local `/etc/hosts` file and similarly checks for the PR (Primary) and OB (Observer) host entries on their respective servers via SSH. These checks ensure that hostname resolution is correctly configured, which is crucial for inter-server communication.

###### Oracle Home and Binaries Verification
- Checks if the `ORACLE_HOME` environment variable is set and if essential binaries like `sqlplus` are present and executable. This verification is conducted both locally and remotely (for the primary server) to ensure the Oracle Database software is correctly installed and accessible.

###### Remote dgmgrl Executable Check
- Specifically checks for the presence of the `dgmgrl` (Data Guard Manager) executable on the observer host. This tool is essential for managing Oracle Data Guard configurations, indicating the observer's readiness to participate in Data Guard operations.

###### Oracle Instance Entry in oratab
- Verifies the presence of the primary database instance in the `/etc/oratab` file on the primary server, confirming that the instance is recognized by the system's Oracle software environment.

###### Oracle Home Version Consistency
- Extracts and compares the detailed Oracle version information from both the primary and standby servers using `sqlplus -v`. This step is crucial to ensure compatibility between the primary and standby instances, as version mismatches can lead to unforeseen issues.

###### TNS Ping Tests
- Conducts `tnsping` tests to check network connectivity to the primary and standby databases from various locations (including remote checks from the primary and observer servers). `tnsping` is a utility used to test the listener configuration for Oracle Database connections.

###### Database Size and Free Space Verification
- Calculates the size of the primary database and compares it against the available free space on the standby server. This check is important to ensure that the standby server has sufficient space to accommodate the primary database in scenarios such as data synchronization or recovery.

###### Summary of Checks
- **Hostname resolution** on local and remote systems to prevent network communication issues.
- **Oracle software installation and configuration** by verifying `ORACLE_HOME` and essential binaries.
- **Network connectivity** between servers using `tnsping`, ensuring that Oracle Net Services are configured correctly.
- **Data Guard prerequisites** by checking for the `dgmgrl` tool and instance entries in `/etc/oratab`.
- **Compatibility and capacity** by comparing Oracle versions and assessing disk space requirements.

###### Output and Logging
- The function redirects its output to a log file, providing a comprehensive record of the pre-checks performed. This approach facilitates troubleshooting and auditing of the database environment's readiness for operations.

6.  The function `check_sqlplus_connection` is designed to verify SQL*Plus connections to Oracle databases, specifically targeting primary and standby databases as part of a broader Oracle Data Guard or similar high-availability setup. It executes SQL*Plus commands to connect to each database using their respective TNS (Transparent Network Substrate) names and handles potential Oracle errors. Here's a detailed breakdown:

###### Functionality Overview
- **Purpose**: To check the ability to connect to primary and standby databases using SQL*Plus and handle various Oracle errors that might arise during the connection attempt.
- **Method**: The function uses here-documents to execute SQL*Plus commands, directing SQL*Plus to exit with the SQL error code (`SQL.SQLCODE`) upon encountering a SQL error (`WHENEVER SQLERROR EXIT SQL.SQLCODE;`).

###### Primary Database Connection Check
**Connection Attempt**: Executes a SQL*Plus command to connect to the primary database as `SYSDBA` and attempts to select a string from the dual table—a dummy table used in Oracle databases for various utility purposes.
**Error Handling**: Checks the output for the presence of "ORA-" error codes. If an error code is detected, it logs an error message indicating a failure to connect, removes a temporary or control file (`$DAT_FL`), and exits with an error status. If no error is found, it logs a success message.

###### Standby Database Connection Check
**Connection Attempt**: Similarly attempts to connect to the standby database.
**Success and Idle Instance Handling**: If the connection is successful and the output does not indicate an "idle instance" status, it logs a success message. For idle instances, a specific message is logged, indicating the database's readiness for further processing.
**Credential Errors and Cleanup**: Checks the exit status (`$sql_error_code`) from the SQL*Plus command. If an error code indicating invalid credentials (`1017`) is encountered, it logs an error message, removes `$DAT_FL`, and exits. Regardless of the specific error, if the database is in an idle state, it proceeds to remove sensitive information.
**Cleanup Actions**: For both primary and standby checks, if certain conditions are met (such as successful connection or specific errors), the function attempts to sanitize the control file (`$CTL_FILE`) by removing or obfuscating sensitive information like Oracle and wallet passwords.

###### Security and Maintenance Considerations

- **Logging and Error Reporting**: Through echoing structured messages and utilizing a custom logging function (`log_script_info`), the script ensures that significant actions and outcomes are logged, aiding in troubleshooting and operational auditing.
- **Exit Strategy**: By employing conditional exits based on error codes and connection statuses, the function ensures that the script halts execution when critical errors are encountered.

7. The script comprises a series of Bash functions designed to automate the preparation and configuration of Oracle Data Guard. Data Guard is a feature of Oracle Database that provides a comprehensive set of services designed to create, maintain, manage, and monitor one or more standby databases to enable disaster recovery and high availability. Let's break down each function and its purpose:

###### `create_pfile` Function
- **Purpose**: Generates an initialization parameter file (`pfile`) and a startup SQL script for an Oracle database instance.
- **Process**:
  - Creates an `init.ora` file with basic database parameters like database name (`db_name`), unique name (`db_unique_name`), and block size (`db_block_size`).
  - Generates a SQL script to shut down the database (if running) and start it in `nomount` mode using the `pfile`. This mode is used for maintenance tasks and initial database creation steps.

###### `create_rman_duplicate_script` Function
- **Purpose**: Prepares an RMAN (Recovery Manager) script for duplicating the primary database to create a standby database.
- **Process**:
  - Creates necessary directories on the standby server for administrative dumps, logs, and the Fast Recovery Area (FRA).
  - Writes an RMAN script to perform the database duplication, adjusting parameters specifically for a standby configuration, including file name conversions and Data Guard-specific settings.

###### `create_dg_setup_primary_file` Function
- **Purpose**: Constructs a SQL script (`dg_setup_primary.sql`) to configure the primary database for Data Guard replication.
- **Process**:
  - Establishes necessary directories on the primary server.
  - The SQL script adjusts various Oracle system settings to enable Data Guard functionality, like defining file destinations, enabling the Data Guard broker, and configuring automatic file management.

###### `create_dg_broker_file` Function
- **Purpose**: Generates a Data Guard Broker configuration file (`dg_broker.dgs`) to manage the Data Guard environment.
- **Process**:
  - Defines the Data Guard configuration, including the primary and standby databases, using the `create configuration` command.
  - Sets properties related to network connectivity, failover behavior, and logging transport mode.

###### `create_dg_checks_file` Function
- **Purpose**: Creates a script (`dg_checks.dgs`) to perform various checks and validations on the Data Guard configuration.
- **Process**:
  - Verifies the network configuration, static connection identifiers, and other critical settings.
  - Validates the primary and standby databases' configurations for consistency and correctness.

###### General Observations
- **Automation and Efficiency**: These functions automate critical steps in setting up Oracle Data Guard, reducing manual effort and the potential for human error.
- **Flexibility**: The use of variables (e.g., `${SCRIPT_LOC}`, `${DR_INSTNAME}`, `${PRIMARY_TNS}`) allows for customization and flexibility, making the script adaptable to different environments.
- **Error Handling**: While the script focuses on configuration generation, it's important to incorporate comprehensive error handling in the broader context where these functions are invoked, especially for operations like remote SSH commands and SQL/RMAN executions.

8. The `primary_configuration` function is designed to prepare the primary database for Oracle Data Guard configuration by updating necessary parameters, ensuring prerequisites like standby redo logs, force logging, and flashback are correctly set up. This function is a crucial step in establishing a robust Data Guard environment, which ensures data protection, high availability, and disaster recovery for Oracle databases. Let's break down the operations performed by this function:

###### Logging Setup
- **Purpose**: Redirects standard output (stdout) and standard error (stderr) to a designated log file. This aids in capturing the output of the script's execution for troubleshooting and auditing purposes.
- **Implementation**: Uses `exec` to redirect output to a file within a specified log directory.

###### Script Existence Check
- Checks for the existence of the `dg_setup_primary.sql` script within a specified location. If the script is not found, it logs an error message, removes a temporary data file (`$DAT_FL`), and exits the function to prevent executing incomplete or incorrect configuration steps.

###### Data Guard Configuration Execution
- Executes the `dg_setup_primary.sql` script using `sqlplus`, an Oracle database command-line tool, to apply necessary Data Guard configuration settings on the primary database. The execution output is logged to a file for review.

###### Standby Redo Logs Check and Creation
- Queries the primary database to check if standby redo logs exist. Standby redo logs are crucial for managing redo data on the standby database and are required for proper Data Guard operation.
- If no standby redo logs are found, it dynamically adds them based on the existing redo log configuration to ensure the standby database can manage redo data effectively.

###### Force Logging Check and Enablement
- Checks if force logging is enabled on the primary database. Force logging ensures all changes are logged, supporting data consistency between the primary and standby databases.
- Enables force logging if it is not already enabled, guaranteeing that all database changes are captured in the redo logs, a necessity for accurate data recovery and synchronization.

###### Flashback Mode Check and Enablement
- Verifies whether flashback mode is enabled on the primary database.
- Enables flashback mode if it is not already active, enhancing the database's ability to recover from errors and aligning with best practices for Data Guard configurations.

9.  The `standby_creation` function is designed to automate the creation of a standby database using Recovery Manager (RMAN) and to enable Flashback  for additional data protection in an Oracle Data Guard configuration. This function encapsulates a series of steps, each critical to ensuring a smooth and reliable setup of the standby database. Here's a detailed explanation of its operations:

###### Setup and Preliminary Checks
- **Output Redirection**: All command output is redirected to a designated log file, facilitating easier troubleshooting and auditing of the standby database creation process.
- **File Existence Verification**: It checks for the necessary initialization parameter file (`init.ora`) and SQL startup script, exiting with an error if either is not found. These files are essential for correctly configuring and starting the standby database in the correct mode.

###### Standby Database State Verification
- **Database State Check**: Before proceeding, the script checks the current state of the standby database to ensure it's not already in a `MOUNTED` or `OPEN` state, as such states would be inappropriate for the creation process.
  
###### Directory Creation
- **Preparation**: Creates required directories for the standby database, particularly for administrative dumps (`adump`), which are crucial for storing diagnostic data and logs.

###### Database Startup in NO MOUNT State
- **NO MOUNT Startup**: Executes the SQL startup script to initiate the standby database in a `NO MOUNT` state, allowing RMAN to duplicate the primary database without interference from existing database structures.

###### RMAN Duplication
- **RMAN Script Execution**: Utilizes RMAN to execute a duplication script, creating the standby database from the primary database. This process is critical for setting up the standby database with a copy of the primary database's data, ensuring data consistency and facilitating seamless failover capabilities.
- **Error Handling**: After the RMAN duplication, it searches the log file for any Oracle (`ORA-`) or RMAN-specific errors, indicating a problem with the duplication process. If errors are found, the function logs an error message, removes a temporary data file, and exits.

###### Enabling Flashback Technology
- **Flashback Activation**: Lastly, the function enables Flashback technology on the standby database, offering an additional layer of data protection by allowing the database to be reverted to a previous state, simplifying recovery from human errors or data corruptions.

###### General Observations
- **Comprehensive Automation**: This function aims to automate the complex and error-prone process of standby database creation, from initial checks to the final setup, including RMAN duplication and Flashback technology enablement.
- **Security and Error Handling**: Through meticulous checks and error handling, the function ensures that each step is completed successfully before proceeding, reducing the risk of misconfiguration or partial setup.
- **Logging and Auditing**: The emphasis on logging throughout the function not only aids in troubleshooting but also provides a clear audit trail of the standby database creation process.

10.  The `dataguard_broker_setup` function is structured to automate the setup and validation of Oracle Data Guard Broker configurations. Data Guard Broker is a management tool provided by Oracle to simplify the setup, management, and monitoring of Oracle Data Guard configurations. Here's a detailed explanation of the process and the components involved in this function:

###### Output Redirection
- **Purpose**: Ensures that all output, including errors, is captured in a log file for auditing and troubleshooting. This is essential for monitoring the setup process and identifying potential issues.

###### Configuration File Verification
- Checks for the presence of Data Guard Broker configuration scripts (`dg_broker.dgs` for setting up the broker and `dg_checks.dgs` for post-setup validation). If either file is missing, it logs an error and exits, preventing partial or incorrect configuration.

###### Standby Database State Verification
- **Nested Function**: `check_standby_state` is defined and used within `dataguard_broker_setup` to verify that the standby database is in an appropriate state (either `MOUNTED` or `OPEN`) for Data Guard Broker operations. This check is critical because certain Data Guard Broker commands require the standby database to be accessible.
- **Error Handling**: If the standby database is not in the required state, the function logs an error and exits. This preemptive check ensures that the environment is correctly prepared for Data Guard Broker setup.

###### Data Guard Broker Setup
- Executes the `dg_broker.dgs` script using `dgmgrl` (Data Guard Manager command-line interface) to configure the Data Guard Broker. This step involves creating configurations, setting up properties, and potentially starting the Data Guard Broker.
- **Output Log**: The output of the `dgmgrl` command is both displayed and appended to a log file, allowing for real-time monitoring and subsequent review.

###### Error Detection in Broker Setup
- Searches the broker setup log for Oracle error patterns (`ORA-`). If errors are detected, the function logs a message, removes any temporary data file, and exits. This step ensures that any issues encountered during the setup process are promptly addressed.

###### Data Guard Broker Validation
- Executes the `dg_checks.dgs` script using `dgmgrl` to perform a series of validation checks on the Data Guard Broker configuration. This includes verifying network configurations, static connect identifiers, and ensuring the overall Data Guard setup is coherent and ready for operation.
- **Validation Log**: Similar to the setup process, the validation output is also tee-d to a log file for review.

###### Successful Completion
- Upon successful execution and validation, the function logs a completion message, indicating the Data Guard Broker setup and validation have been successfully completed.

11. The `dataguard_verification` function, complemented by the `dgverify.sql` PL/SQL script, aims to conduct an in-depth verification of an Oracle Data Guard configuration. This script performs a series of checks to ensure the Data Guard environment is correctly configured and operational. Here’s a breakdown of the script’s functionality and the checks it performs:

###### Initial Setup
- **Output Redirection**: All command outputs, including errors, are redirected to a specified log file, facilitating effective logging and auditing of the verification process.
- **Verification Script Existence Check**: Ensures the `dgverify.sql` script is present before proceeding, ensuring the verification process can be executed.

###### Verification Checks Execution
- Executes the `dgverify.sql` script against both the primary and standby databases using SQL*Plus. This script assesses various aspects of the Data Guard configuration to ensure its integrity and operational readiness.

###### Log File Analysis
- Analyzes the output log for the word "ERROR" to identify any issues discovered during the verification checks. If errors are detected, the script logs an error message, performs cleanup, and exits.

###### Detailed PL/SQL Script Analysis (`dgverify.sql`)
- **Dynamic Configuration Checks**: Utilizes dynamic performance views and undocumented or specialized views (e.g., `x$drc`) to gather configuration details and status information about the Data Guard setup.
- **Database Role and Status Verification**: Confirms the database role (primary or standby) and checks if Data Guard is enabled and functioning correctly.
- **Connectivity Checks**: Verifies the reachability of the primary database from the standby perspective and vice versa, ensuring that network communications between the databases are intact.
- **Flashback Status**: Checks whether Flashback is enabled, offering an additional layer of data protection.
- **Apply and Transport Lag Measurement**: Measures the apply and transport lag to ensure data changes are being replicated and applied within acceptable thresholds, critical for maintaining data consistency and minimizing data loss in failover scenarios.
- **Error and Warning Summary**: Aggregates the results of the checks into a summary of errors and warnings, providing a clear indication of the Data Guard configuration's health.

###### PL/SQL Script Execution Flow
- **Cursor Definitions**: Defines cursors to fetch Data Guard configuration details, including database roles, intended states, and connectivity information.
- **Configuration and Lag Checks**: Performs detailed checks on the configuration status, primary database reachability, log transport status, and lag metrics.
- **Outcome Evaluation**: Concludes with a comprehensive evaluation of the Data Guard environment's health, categorizing the findings into errors and warnings to guide further action.

12.  The `set_MaxAvailability` function is designed to update the protection level of an Oracle Data Guard configuration to Maximum Availability mode. This function uses Oracle Data Guard Broker's command-line interface (`dgmgrl`) to make the necessary changes. Here’s an overview of the steps and commands involved:

###### Log File Redirection
- **Purpose**: Ensures that all output from the function, including any errors, is captured in a log file specified by `${SCRIPT_LOC}/logs/${PRG}_$cdt.log`. This approach facilitates debugging and auditing of the operation.

###### Start of Process Notification
- Notifies the start of the process to update the Data Guard protection level to Maximum Availability, indicating the function's intention to modify the Data Guard configuration.

###### Data Guard Broker Connection
- Establishes a connection to the Oracle Data Guard Broker using `dgmgrl` and the primary database's TNS alias (`${PRIMARY_TNS}`). Oracle Data Guard Broker is a management and monitoring tool provided by Oracle to simplify the management of Data Guard configurations.

###### Commands Executed in Data Guard Broker
**Set Log Transport Mode to Synchronous**:
 `EDIT DATABASE ${DR_DGBR_NAME} SET PROPERTY LogXptMode='SYNC';`
 - This command configures the log transport mode for the standby database (`${DR_DGBR_NAME}`) to synchronous (`SYNC`), ensuring that no data loss occurs during a failover. In this mode, a transaction is not considered committed until all redo necessary to recover the transaction has been written to both the local online redo log and the standby redo log on at least one synchronous standby database.
 `edit database ${PR_DGBR_NAME} set property LogXptMode='SYNC';`
 - Similarly, sets the primary database's log transport mode to synchronous. The case difference (`EDIT` vs. `edit`) does not affect the execution; Oracle Data Guard Broker commands are case-insensitive.
**Update Protection Mode to Maximum Availability**:
 `EDIT CONFIGURATION SET PROTECTION MODE AS MaxAvailability;`
 - This command changes the Data Guard configuration's protection mode to Maximum Availability. This mode provides the highest level of data protection that is possible without compromising the availability of the primary database. It ensures that no data loss occurs if the primary database fails, as long as a synchronous standby database is available and can receive redo data.
**Show Configuration**:
 `SHOW CONFIGURATION;`
 - Displays the current Data Guard configuration, including databases, their roles (primary or standby), transport modes, and the protection mode. This command is used here to verify that the changes have been applied successfully.

13. The `enable_Fast_Start_Failover` function is designed to configure and enable Fast-Start Failover (FSFO) within an Oracle Data Guard configuration using the Oracle Data Guard Broker's command-line interface, `dgmgrl`. Fast-Start Failover allows Data Guard to automatically fail over to a standby database without manual intervention, improving availability by reducing downtime in the event of a primary database failure. Here's a step-by-step explanation of the function:

###### Output Redirection
- **Purpose**: All command outputs, including errors, are redirected to a log file specified by `${SCRIPT_LOC}/logs/${PRG}_$cdt.log`. This facilitates tracking and auditing the Fast-Start Failover enablement process.

###### Notification of Process Start
- The process begins with a notification that Fast-Start Failover is being enabled for the Data Guard configuration, providing a clear indication of the operation's commencement.

###### Execution of `dgmgrl` Commands
- The function connects to the Oracle Data Guard Broker using `dgmgrl` and the primary database's TNS alias (`${PRIMARY_TNS}`). It then executes several commands within a here-document (`EOF` block), detailed as follows:
- **SHOW FAST_START FAILOVER**: Displays the current Fast-Start Failover configuration.
- **EDIT DATABASE** commands: Sets the Fast-Start Failover target for both the primary (`${PR_DGBR_NAME}`) and standby (`${DR_DGBR_NAME}`) databases to each other. This configuration ensures that each database is aware of its counterpart for failover purposes.
- **EDIT CONFIGURATION SET PROPERTY FastStartFailoverThreshold**: Configures the Fast-Start Failover threshold (`${FSFO_THRHLD}`), which is the maximum time (in seconds) the observer is allowed to attempt a failover before declaring the primary database as unavailable.
- **SHOW CONFIGURATION** commands: Various `SHOW CONFIGURATION` commands are used to display the current settings related to Fast-Start Failover, including parameters for primary shutdown, automatic reinstatement after failover, and the failover threshold.
- **ENABLE FAST_START FAILOVER**: Enables Fast-Start Failover within the Data Guard configuration, activating the automated failover capability.

###### Confirmation of Completion
- After successfully executing the `dgmgrl` commands, the function logs a confirmation message indicating that Fast-Start Failover has been enabled successfully.

14.  The `start_observer` function is designed to initiate the Oracle Data Guard Observer, a component crucial for the implementation of the Fast-Start Failover capability within an Oracle Data Guard configuration. The Observer automates the failover process to the standby database when the primary database becomes unavailable, ensuring minimal disruption and maintaining high availability. Below is a detailed explanation of how this function operates:

###### Log File Redirection
- **Purpose**: At the beginning of the function, all standard output and errors are redirected to a specific log file (`${SCRIPT_LOC}/logs/${PRG}_$cdt.log`). This ensures that all operations performed by this function, including any issues encountered during the Observer startup, are captured for auditing and troubleshooting purposes.

###### Notification of Process Start
- The function logs the initiation of the Data Guard Observer start process using a predefined logging function or mechanism (`log_script_info`). This provides clear feedback about the operation's commencement, which is crucial for operational transparency and monitoring.

###### Observer Startup Command Execution
- Utilizes `dgmgrl`, the Data Guard Manager command-line interface, to execute the "start observer" command. This command is issued against the standby database's TNS alias (`${STANDBY_TNS}`), indicating where the Observer should monitor the status of the Data Guard configuration.
- The `nohup` command is used to run the Observer in the background, ensuring that the process continues to run even if the session from which it was started is closed. The output of the Observer process is directed to a dedicated log file (`observer_sanofiobs.log`) within the specified script location (`${SCRIPT_LOC}`). This setup allows for continuous monitoring without requiring an open terminal session.
- The use of `>>` for redirecting output appends any new information to the `observer_sanofiobs.log` file, rather than overwriting it, preserving the history of Observer activity.

15. The `stop_observer` function is crafted to halt the operation of the Oracle Data Guard Observer, a critical component for managing Fast-Start Failover (FSFO) within an Oracle Data Guard setup. The Observer's role is to monitor the availability of the primary database and trigger a failover to the standby database automatically if the primary becomes unavailable. Here's a detailed breakdown of the function:

###### Log File Redirection
- **Purpose**: All standard output and errors produced during the execution of the function are redirected to a designated log file (`${SCRIPT_LOC}/logs/${PRG}_$cdt.log`). This redirection is crucial for capturing the details of the Observer stop process, facilitating subsequent auditing and troubleshooting if necessary.

###### Process Initiation Notification
- The function begins with a notification about the start of the Observer termination process. This message serves to inform any administrators or automated monitoring systems that an operation affecting the Data Guard environment's failover capabilities is underway.

###### Execution of `dgmgrl` Command
- Utilizes Oracle's Data Guard Manager command-line interface (`dgmgrl`) to issue the command to stop the Observer. The command is executed in the context of the standby database's TNS alias (`${STANDBY_TNS}`), directing the specific Data Guard environment where the Observer should be stopped.
- The use of `nohup` ensures that the command to stop the Observer runs in the background, allowing the operation to continue even if the initiating session is closed. Output from this operation is appended to a dedicated Observer log file (`observer_sanofiobs.log`), located within the script's specified directory (`${SCRIPT_LOC}`). This approach ensures that the operation's details are captured without requiring the session to remain open.

16. The function `execute_dg_info_sql_pre` is designed to execute a comprehensive diagnostic script (`dg_info.sql`) against an Oracle database involved in a Data Guard setup, prior to a switchover operation. This function captures a snapshot of the Data Guard environment's status and configuration, spooling the output to a log file for review. Below is an explanation of the function and the diagnostic script it runs:

###### Function Workflow:
**Log File Redirection**: The function begins by redirecting all output, including errors, to a log file specified in the script location path variable (`${SCRIPT_LOC}/logs/`). This ensures that the execution output is captured for auditing and troubleshooting purposes.
   
**Notification**: It announces the initiation of the Data Guard Observer diagnostic process, providing transparency about the operation being performed.
   
**Execution of Diagnostic SQL Script**: The function executes `dg_info.sql` via SQL Plus, an Oracle command-line utility that allows SQL and PL/SQL commands to be executed against a database.
   
**Spooling Output**: The output of `dg_info.sql` is spooled to a log file, `before_switchover_verbose_$dt.log`, capturing the state of the Data Guard environment before any switchover activity. This file is stored in the specified script location path, providing a timestamped record.

###### Diagnostic Script (dg_info.sql) Overview:
The `dg_info.sql` script performs a thorough examination of the Data Guard configuration and the database's operational parameters. It is structured to collect data on various aspects of the Data Guard setup and the database's health:

- **Database Role and Status**: Queries the `v$database` view for fundamental database characteristics, including the database role (primary or standby), the protection mode, and the current operation mode.

- **Data Guard Configuration**: Extracts information from `v$dataguard_config` to understand the Data Guard topology, including primary and standby databases and their roles.

- **Archive Destination Status**: Gathers details from `v$archive_dest` and `gv$archive_dest_status` about the archive log destinations, their status, and any errors encountered in shipping logs.

- **Database Incarnation**: Reviews the database's incarnation history from `v$database_incarnation` to understand the database's resetlogs history, which is crucial for recovery and failover operations.

- **Instance and Parameter Settings**: Collects data from `gv$instance` and `gv$parameter` on instance-level settings and parameters that influence Data Guard operations, such as log archive destinations and Data Guard-specific settings.

- **Data Guard Process Activity**: Reports on the activity and status of Data Guard processes from `v$managed_standby`, providing insights into the real-time apply and transport mechanisms.

- **Apply Lag and Archive Gaps**: Assesses apply lag and potential archive gaps using `v$dataguard_stats` and `v$archive_gap`, which are critical for maintaining data consistency across the Data Guard configuration.

- **Session and Activity Counts**: Concludes with counts of active sessions and total sessions from `v$session`, giving an overview of the database's workload at the time of the report.

17. - The `perform_switchover` function is structured to execute a switchover operation within an Oracle Data Guard environment. A switchover is a controlled role transition where the primary database becomes a standby, and a standby database becomes the primary, without data loss. This operation is essential for planned maintenance or testing disaster recovery readiness without impacting data integrity. Here’s a breakdown of the function’s workflow:

###### Log File Redirection
- **Purpose**: Ensures all outputs, including errors, from the function's execution are captured in a designated log file. This is critical for auditing, troubleshooting, and historical reference.

###### Preliminary Steps
- **`execute_dg_info_sql_pre`**: Executes a diagnostic script to capture the state of the Data Guard configuration before the switchover. This step is vital for baseline documentation and can help in verifying post-switchover success and troubleshooting any issues.
  
- **`dataguard_verification`**: Runs verification checks on the Data Guard configuration to ensure that the environment is in a healthy state before attempting the switchover. This proactive step is crucial to avoid complications during the switchover process.

###### Session Count Queries
- Executes two SQL queries to count the total active sessions and the total number of sessions in the database. This information can be used to assess the database's workload before proceeding with the switchover, ensuring minimal disruption.

###### Switchover Execution
- **DGMGRL Commands**:
  - Connects to the Data Guard Broker using the standby database's TNS alias (`${STANDBY_TNS}`).
  - **`show configuration`**: Displays the current Data Guard configuration, providing a snapshot of the system's state before the switchover.
  - **`switchover to ${DR_DGBR_NAME}`**: Initiates the switchover to the designated standby database (`${DR_DGBR_NAME}`). This command is the core of the function, where the roles of the primary and standby databases are swapped.
  - **`show configuration`**: Re-displays the Data Guard configuration post-switchover to confirm the operation's success and validate the new roles of the databases.

###### Post-Switchover Diagnostic
- The function includes a commented-out call to `execute_dg_info_sql_post` (not executed due to being commented out), which suggests an intention to capture the state of the Data Guard configuration after the switchover.

###### Function Identification
- Sets `current_function="perform_switchover"` towards the end, which seems to be intended for logging or tracking purposes but is positioned after most operations are already called. For better clarity and tracking, this should ideally be set at the beginning of the function.

18. - The `perform_switchback` function is designed to revert a previously executed switchover in an Oracle Data Guard configuration, effectively switching roles back between the primary and standby databases. This process is typically used after maintenance tasks or testing procedures on the original primary database have been completed and it's ready to resume its role as the primary. The function aims to ensure a seamless transition back to the original configuration with minimal disruption. Here’s an overview of the function's workflow:

###### Log File Redirection
- **Purpose**: Similar to the switchover function, all output, including errors from the function's execution, is redirected to a log file. This step is essential for capturing the execution details for auditing and troubleshooting purposes.

###### Preliminary Diagnostic and Verification
- **`execute_dg_info_sql_pre`**: Runs a diagnostic script to capture the state of the Data Guard environment before the switchback operation. This step is crucial for baseline documentation and can assist in post-operation validation.
  
- **`dataguard_verification`**: Conducts checks on the Data Guard configuration to ensure its readiness for the switchback operation. This step helps to prevent potential issues during the role transition by ensuring the environment's health.

###### Session Activity Queries
- Executes SQL queries to count the total number of active sessions and the total number of sessions within the database. This information provides insight into the database's workload and activity level before proceeding with the switchback, which can be helpful for planning the operation to minimize impact.

###### Switchback Execution using DGMGRL
- **Connection**: Uses `dgmgrl` to connect to the Data Guard Broker, this time using the primary database's TNS alias (`${PRIMARY_TNS}`), reflecting that the original standby (now acting as the primary) will be involved in the switchback operation.
- **Commands**:
  - **`show configuration`**: Displays the current Data Guard configuration for an overview of the environment prior to the switchback.
  - **`switchover to ${PR_DGBR_NAME}`**: Executes the switchover command targeting the original primary database's Data Guard Broker name (`${PR_DGBR_NAME}`). This action reverses the roles, making the original primary (which was acting as a standby post-switchover) the primary again.
  - **`show configuration`**: Re-checks the Data Guard configuration post-switchback to confirm the operation's success and verify the roles of the databases have been successfully swapped back.

###### Post-Switchback Diagnostic
- Similar to the switchover function, a commented-out call to `execute_dg_info_sql_post` suggests an intended follow-up diagnostic step to capture the Data Guard configuration state after the switchback operation. 

###### `configure_prerequisites()` Function Summary
- **SSH Connectivity Checks**: Verifies SSH connectivity between primary, standby, and observer hosts to ensure network communication prerequisites are met.
- **`create_password_file`**: Generates a password file for database authentication, facilitating secure connections.
- **`configure_listener_tns`**: Sets up the listener on the standby server and creates TNS entries on primary, standby, and observer servers for network configuration.
- **`create_wallet`**: Initiates an Oracle wallet on the standby server for secure credential storage.
- **`add_wallet`**: Adds credentials to the Oracle wallet on the standby server and distributes wallet files to the primary and observer servers for secure authentication.
- **`create_pfile`**: Generates a parameter file (pfile) and a startup script for initializing the Oracle database instance, essential for database creation and startup.
- **`create_dg_setup_primary_file`**: Produces a SQL script for setting required parameters on the primary server for Data Guard, aiding in environment preparation.
- **`create_rman_duplicate_script`**: Develops an RMAN script for duplicating the primary database to establish the standby database, a crucial step for Data Guard setup.
- **`create_dg_broker_file`**: Creates a configuration file for Data Guard Broker setup, facilitating Data Guard management.
- **`create_dg_checks_file`**: Generates a checks file for verifying the Data Guard Broker configuration, ensuring the integrity and correctness of the setup.
- **`perform_pre_checks`**: Conducts initial checks including hostname verification, ORACLE_HOME binaries and version, primary database's oratab entry, and tnsping, ensuring system readiness.
- **`check_sqlplus_connection`**: Reiterates the verification of SQL*Plus connectivity without a password for operational security and ease of use.

###### `dg_configuration()` Function Summary
- **`check_sqlplus_connection`**: Verifies SQL*Plus connectivity without a password for enhanced security and convenience.
- **`primary_configuration`**: Applies necessary parameter updates on the primary database for Data Guard configuration.
- **`standby_creation`**: Automates the creation of the standby database using RMAN and enables Flashback Technology for data recovery.
- **`dataguard_broker_setup`**: Configures the Data Guard Broker for managing the Data Guard configuration.
- **`dataguard_verification` (commented out)**: Executes verification checks to ensure Data Guard is correctly configured and operational.

19. The script segment utilizes a `case` statement to manage various operations related to Oracle Data Guard configuration and management. This structure allows for a modular approach to executing distinct functions based on the command-line argument (`$1`) provided when the script is called. Here's a simplified overview of each case option and its associated command:

##### Case Options and Commands

- **`configure_prerequisites`**: Prepares the environment by verifying prerequisites and creating necessary files before configuring Data Guard.
  
- **`pre_checks`**: Conducts initial checks, including hostname verification and SQL*Plus connectivity.

- **`dg_configuration`**: Executes the complete Data Guard configuration process, including primary and standby setups, and Data Guard Broker configuration.

- **`primary_configuration`**: Applies necessary parameter updates to the primary database for Data Guard setup.

- **`standby_creation`**: Automates the standby database creation using RMAN and enables flashback technology.

- **`dgbroker_setup`**: Configures the Data Guard Broker for easier management of the Data Guard environment.

- **`set_MaxAvailability`**: Updates the Data Guard protection level to Maximum Availability to ensure high data protection without compromising availability.

- **`enable_Fast_Start_Failover`**: Enables Fast-Start Failover to allow automatic failover to the standby database in case of primary database failure.

- **`start_observer`**: Initiates the Data Guard Observer process for monitoring and managing Fast-Start Failover.

- **`stop_observer`**: Stops the Data Guard Observer process.

- **`dgverify`**: Performs verification checks to ensure the Data Guard configuration is correct and complete.

- **`create_wallet`**: Creates a new Oracle wallet for secure storage of credentials necessary for Data Guard operations.

- **`configure_listener_tns`**: Sets up the network configuration, including the listener on the standby server and TNS entries for primary, standby, and observer servers.

- **`add_wallet`**: Adds credentials to the Oracle wallet and verifies SQL*Plus connectivity.

- **`switchover`**: Initiates a controlled switchover between the primary and standby databases.

- **`switchback`**: Reverses the roles of the primary and standby databases back to their original state after a switchover.

- **`*` (default case)**: Handles invalid options by displaying a usage message with all available commands and exiting the script.

###### Conclusion
This script provides a comprehensive framework for managing an Oracle Data Guard environment through a series of targeted operations. Each case within the `case` statement corresponds to a specific function that addresses different aspects of Data Guard setup and maintenance, from initial configuration to advanced features like Fast-Start Failover and switchover management. This structure not only enhances script readability and maintainability but also simplifies the execution of complex Data Guard administration tasks.
