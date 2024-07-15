# Upgrade using Replay Upgrade #

## Introduction ##

 This lab will demonstrate how easy it is to unplug an existing Pluggable Database and plug it into an Oracle Home that has either been patched or upgraded to a new version.

 Estimated time: 45 minutes

### Objectives ###

- Unplug the source PDB from its current database
- Plug the PDB into its new CDB running on 23ai
- Wait until the upgrade was done automatically

### Prerequisites ###

- You have access to the Upgrade to the Upgrade to 23ai Livelab image
- A new 23ai database has been created in this image (from Lab 1)
- All databases in the image are running

When in doubt or need to start the databases use the following steps:

1. Please log in as **oracle** user and execute the following command:

    ```text
    $ <copy>. oraenv</copy>
    ```

2. Please enter the SID of the 23ai database that you have created in the first lab. In this example, the SID is **`DB23AI`**

    ```text
    ORACLE_SID = [oracle] ? <copy>DB23AI</copy>
    The Oracle base has been set to /u01/oracle
    ```
3. Now execute the `dbstart` command to start all databases listed in the `/etc/oratab` file:

    ```text
    $ <copy>dbstart $ORACLE_HOME</copy>

    Processing Database instance "BBB": log file /u01/oracle/product/19/dbhome/rdbms/log/startup.log
    Processing Database instance "CCC": log file /u01/oracle/product/19/dbhome/rdbms/log/startup.log
    Processing Database instance "RRR": log file /u01/oracle/product/19/dbhome/rdbms/log/startup.log
    Processing Database instance "TTT": log file /u01/oracle/product/19/dbhome/rdbms/log/startup.log
    Processing Database instance "DB23AI": log file /u01/oracle/product/23/dbhome/rdbms/log/startup.log
    ```
    
## Task 1: Shutdown the source PDB ##

 Please login as the **oracle** user to check the status of the source database and pluggable databases.

### Set the environment for the 19c RRR database ###

1. Set the environment to the correct ORACLE\_HOME and ORACLE\_SID:

    ```text
    $ <copy>. oraenv</copy>
    ```
    Enter the database 19c SID when requested:

    ```text
    ORACLE_SID = [oracle] ? <copy>RRR</copy>
    The Oracle base remains unchanged with value /u01/oracle
    ```

2. Now we can log in as sysdba and check the status of the database.

    ```text
    $ <copy>sqlplus / as sysdba</copy>

    SQL*Plus: Release 19.0.0.0.0 - Production on Tue Jun 25 13:15:50 2024
    Version 19.21.0.0.0

    Copyright (c) 1982, 2022, Oracle.  All rights reserved.

    Connected to:
    Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
    Version 19.21.0.0.0
    ```
    
3. Check the status of the PDB called RRR01 in this container database

    ```text
    SQL> <copy>show pdbs</copy>

    CON_ID CON_NAME     OPEN MODE  RESTRICTED
    ------ ------------ ---------- -----------
        2  PDB$SEED     READ ONLY  NO
        3  RRR01        MOUNTED
    ```

    >	If the state of the RRR01 Pluggable Database is not *MOUNTED* then shutdown the PDB using the following command:
    >
    ```text
    SQL> <copy>alter pluggable database RRR01 close;</copy>
    
    >Pluggable database altered.
    ```
    
    In the database RRR, we have 1 pluggable database, RRR01 in MOUNTED state, that we can upgrade to 23ai.

## Task 2: Unplugging the source pluggable database ##

 There are many ways to migrate a PDB to a new CDB. Some will keep the data files in place, and other options will recreate the data files (for which you need twice your databasesize available in your storage system). If you are moving the data files to another storage location, you can, for example, also use the Pluggable Archive option (this option will put all files into one zip file with a .pdb extension).

 In this lab, we will keep the data files on the same physical system but we will move the data files to a more logical location. Moving the data files can be done using a single command from SQL*Plus. After the migration to the new location, we can upgrade the PDB.

1. For the following steps, we assume you are already connected as sysdba to the RRR Container Database as described in the previous chapter.

    Execute the following command to unplug the PDB and write a .xml descriptor file to a filesystem location.

    ```text
    SQL> <copy>alter pluggable database RRR01 unplug into '/u01/RRR01.xml';</copy>

    Pluggable database altered.
    ```

2. We will now disconnect from the source database so we can continue importing the PDB.

    ```text
    SQL> <copy>exit</copy>

    Disconnected from Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.21.0.0.0
    ```

## Task 3: Plug the PDB into the target 23ai environment ##

1. First, we need to change the environment settings to the 23aienvironment:

    ```text
    $ <copy>. oraenv</copy>
    ```

    Please enter the SID of the 19c database when asked:

    ```text
    ORACLE_SID = [RRR] ? <copy>DB23AI</copy>
    The Oracle base remains unchanged with value /u01/oracle
    ```

2. We can now login with SQL*Plus as sysdba to execute the import of the PDB:

    ```text
    $ <copy>sqlplus / as sysdba</copy>

    SQL*Plus: Release 23.0.0.0.0 - Production on Tue Jun 25 13:28:05 2024
    Version 23.5.0.24.06

    Copyright (c) 1982, 2024, Oracle.  All rights reserved.


    Connected to:
    Oracle Database 23ai Enterprise Edition Release 23.0.0.0.0 - Production
    Version 23.5.0.24.06
    ```

3. After connecting as sysdba to the target database, we can plug in the PDB. As part of the plug-in process, we can also move (or copy) the data files if needed. In our example, we want the database files to move from the old 19c datafiles location to the new 23ai datafile location:

    ```text
    SQL> <copy>create pluggable database RRR01 using '/u01/RRR01.xml'
         move
         file_name_convert = ('/RRR/','/DB23AI/');</copy>

    Pluggable database created.
    ```

    The `move` clause means that SQL*Plus will move all relevant files to the new location. Using the `file_name_convert`, you can determine what the new location should be.

4. Check that our data files are stored in the 19c datafile location:

    ```text
    SQL> <copy>select name from v$datafile
         where name like '%RRR01%';</copy>

    NAME
    -------------------------------------------
    /u01/oradata/DB23AI/RRR01/system01.dbf
    /u01/oradata/DB23AI/RRR01/sysaux01.dbf
    /u01/oradata/DB23AI/RRR01/undotbs01.dbf
    /u01/oradata/DB23AI/RRR01/users01.dbf
    ```

## Task 4: Open the imported Pluggable Database for Replay Upgrade##

 By opening the 19c PDB in a 23ai CDB, the system will recognize that an upgrade is needed. Instead of opening the pluggable database right away, the system will open the PDB in Upgrade mode automatically and start executing the SQL statements to upgrade the database.
 
 Be aware that the open command will seem to hang until the upgrade has been completed. Depending on the layout of the database, this could take a while. In our example database, it usually takes about 15 minutes.

1. To upgrade the PDB, simply open it in read-write mode:

    ```text
    SQL> <copy>alter pluggable database RRR01 open;</copy>
    ```
    
    This command will seem to hang for about 15 minutes. 
    
2. While the upgrade is running, you can open another (terminal) session and check the progress of the upgrade in the alert log of the database. Please note that the first messages will only appear about 6 minutes after the alter command has been started:

    ```text
    $ <copy>tail -f /u01/oracle/diag/rdbms/db23ai/DB23AI/trace/alert_DB23AI.log</copy>
    
    ...
    RRR01(6):SERVER ACTION=UPGRADE id=: Upgraded from 19.21.0.0.0 to 23.5.0.24.06 Container=RRR01 Id=6
    RRR01(6):SERVER COMPONENT id=DBRESTART: timestamp=2024-06-25 13:48:15 Container=RRR01 Id=6
    RRR01(6):SERVER COMPONENT id=POSTUP_BGN: timestamp=2024-06-25 13:48:15 Container=RRR01 Id=6
    RRR01(6):SERVER COMPONENT id=CATREQ_BGN: timestamp=2024-06-25 13:48:15 Container=RRR01 Id=6
    RRR01(6):SERVER COMPONENT id=CATREQ_END: timestamp=2024-06-25 13:48:27 Container=RRR01 Id=6
    RRR01(6):SERVER COMPONENT id=POSTUP_END: timestamp=2024-06-25 13:48:27 Container=RRR01 Id=6
    RRR01(6):SERVER COMPONENT id=CATUPPST: timestamp=2024-06-25 13:48:28 Container=RRR01 Id=6
    ...
    ```
3. When the upgrade has finished, you will see the regular result when opening a pluggable database:
    
    ```text
    Pluggable database altered.
    ```
    
4. In order to synchronise the CDB with the PDB and to make sure everything starts properly after a reboot, stop and start the new Pluggable Database:

    ```text
    SQL> <copy>alter pluggable database RRR01 close;</copy>
    
    Pluggable database altered.
    ```
    
    and 
    
    ```text
    SQL> <copy>alter pluggable database RRR01 open;</copy>
    
    Pluggable database altered.
    ```
        
5. We can now check if there are any plugin issues for this new pluggable database:

    ```text
    SQL> <copy>select name, cause, message           
         from PDB_PLUG_IN_VIOLATIONS                                                                      
         where status <> 'RESOLVED'
         and con_id = (select con_id from dba_pdbs where PDB_NAME='RRR01');</copy>

    no rows selected
    ```

6. We can also check for any invalid objects in the database:

    ```text
    SQL> <copy>alter session set container=RRR01;</copy>

    Session altered.
    ```

    ```text
    SQL> <copy>select COUNT(*) FROM obj$ WHERE status IN (4, 5, 6);</copy>

      COUNT(*)
    ----------
         10612
    ```

7. If there are any invalid objects, you can recompile them using the `utlrp.sql` script:

    ```text
    SQL> <copy>@$ORACLE_HOME/rdbms/admin/utlrp.sql</copy>

    Session altered.

    Elapsed: 00:00:00.00

    TIMESTAMP
    --------------------------------------------------------------------------------
    COMP_TIMESTAMP UTLRP_BGN	      2024-06-25 14:00:16

    DOC>   The following PL/SQL block invokes UTL_RECOMP to recompile invalid
    DOC>   objects in the database. Recompilation time is proportional to the
    DOC>   number of invalid objects in the database, so this command may take

    <....>

    OBJECTS WITH ERRORS
    -------------------
                      1

    Elapsed: 00:00:00.01
    DOC> The following query reports the number of exceptions caught during
    DOC> recompilation. If this number is non-zero, please query the error
    DOC> messages in the table UTL_RECOMP_ERRORS to see if any of these errors
    DOC> are due to misconfiguration or resource constraints that must be
    DOC> fixed before objects can compile successfully.
    DOC> Note: Typical compilation errors (due to coding errors) are not
    DOC>	   logged into this table: they go into DBA_ERRORS instead.
    DOC>#

    ERRORS DURING RECOMPILATION
    ---------------------------
                              1
    ```
    
    Due to bug 33523831, a new procedure with underlying tables was introduced as a temporary measure to solve an issue with object IDs. The issue has been solved in 23ai so although this procedure is not needed, it will not be removed using the current scripts. Please discard this single invalid object called SYS.OBJNUM\_REUSE\_HOLES in your upgraded PDB.
   
    Your database is now migrated to a new $ORACLE_HOME and upgraded.

This is the end of this lab, you may now continue to the next lab. Do not forget to check the outcome of the Autoupgrade lab (if you had it running while doing this lab).

## Acknowledgements ##

- **Author** - Robert Pastijn, Database Product Management, PTS EMEA - July 2024
