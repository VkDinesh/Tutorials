---
title: Configure the SAP HANA TensorFlow Integration (SAP HANA EML)
description: Provide details on the installation and configuration of the SAP HANA External Machine Learning Library with SAP HANA, express edition.
auto_validation: true
primary_tag: products>sap-hana\, express-edition
tags: [ tutorial>intermediate, topic>cloud, topic>machine-learning, products>sap-hana\, express-edition, products>sap-hana ]
time: 15
---

## Details
### You will learn
- Download, install and configure the TensorFlow integration with SAP HANA, express edition
- Configure your SAP HANA, express edition instance to connect to your TensorFlow Serving instance on Amazon ECS

[ACCORDION-BEGIN [Step 1: ](Access your SAP HANA, express edition instance)]

You can either connect to your SAP HANA, express edition using an SSH client (as described in [Set up SAP HANA, express edition on Amazon Web Services](hxe-xsa-aws-setup)) or use your Jupyter Notebook instance.

If you want to use your Jupyter Notebook instance, open a new Terminal using the menu bar **File > New > Terminal**.

Then run the following command which assumes that you have uploaded your private key file (PEM file) in the SageMaker NoteBook:

```shell
ssh -i <path to pem file>/<pem file name> ec2-user@<instance public IP address>
```

Once you have established an SSH session, the **`ec2-user`** user is logged in and the terminal prompt should be:

```
ec2-user@hxehost:~>
```

Switch now to user **`hxeadm`** to start configuring your instance:

```ssh
sudo su - hxeadm
```

[DONE]
[ACCORDION-END]

[ACCORDION-BEGIN [Step 1: ](SAP HANA External Machine Learning AFL)]

The integration of TensorFlow with SAP HANA is based on the SAP HANA Application Function Library (AFL).

This allows the application developer to elegantly embed TensorFlow function definitions and calls within `SQLScript` and submit the entire code as part of a query to the database.

![SAP HANA EML](00-0.png)

The figure above shows the main components of the integrated solution:

- ***AFL Framework***:
     Allows predefined TensorFlow models to be remotely invoked through `gRPC` calls encapsulated inside AFL procedures
- ***EML AFL***:
     The TensorFlow Serving client implementation for SAP HANA
- ***TensorFlow Serving Server***:
     Makes TensorFlow exported models accessible for execution through `gRPC` remote procedure calls
- ***Active Models***:
    The models currently served and therefore available for execution
- ***`gRPC` Server***:
    The `gRPC` server interface for communication with the TensorFlow Serving `ModelServer` client
- ***Model Persistence***:
    The exported models persisted in a format in a given TensorFlow Serving `ModelServer`

To confirm that the SAP HANA EML functions were installed successfully, you can check the following public views:

- `sys.afl_areas`
- `sys.afl_packages`
- `sys.afl_functions`

Using the **HXE** connection with the **`ML_USER`** user credentials, execute the following SQL statement:

```SQL
SELECT * FROM "SYS"."AFL_AREAS" WHERE AREA_NAME = 'EML';
SELECT * FROM "SYS"."AFL_PACKAGES" WHERE AREA_NAME = 'EML';
SELECT * FROM "SYS"."AFL_FUNCTIONS" WHERE AREA_NAME = 'EML';
```

As a reminder, you can connect using the SAP Web IDE or using HDBSQL with the following command (assuming you didn't change the `ML_USER` password):

```shell
hdbsql -n localhost:39015 -u ML_USER -p Welcome19Welcome19
```

If there result is empty, it probably mean that you need to complete the following tutorial:

- [Install the SAP HANA External Machine Learning Library Package for SAP HANA, express edition](hxe-ua-eml-binary).

The `AFL_AREAS` & `AFL_PACKAGES` should return 1 row each, and the `AFL_FUNCTIONS` should return 10 rows.

[DONE]
[ACCORDION-END]

[ACCORDION-BEGIN [Step 1: ](Configure SAP HANA External Machine Learning)]

Now, that the TensorFlow Serving is up and running in Amazon ECS, you will need to add its configuration to your SAP HANA, express edition instance.

For operational systems when configuring your SAP HANA system, it is highly recommended to use two separate type of users with distinctive roles:

- Administering the TensorFlow Serving and model deployments
- Calling the deployed models in your code

Each type of user will require dedicated roles to be granted.

In your case, you will be reusing the `ML_USER` created during the [Prepare your SAP HANA, express edition instance for Machine Learning](mlb-hxe-setup-basic) tutorial for both roles.

Make also sure that the Script Server has been enabled for your instance.

Using the **HXE** connection with the **SYSTEM** user credentials, execute the following SQL statement:

```SQL
GRANT AFL__SYS_AFL_EML_EXECUTE TO ML_USER;
GRANT SELECT, UPDATE, DELETE, INSERT ON  _SYS_AFL.EML_MODEL_CONFIGURATION TO ML_USER;
```

You can now proceed with the rest of the configuration as `ML_USER`.

Using the **HXE** connection with the **`ML_USER`** user credentials, execute the following SQL statement after adjusting the **ECS Container IP address**:

```SQL
CREATE REMOTE SOURCE "TensorFlow" ADAPTER "grpc" CONFIGURATION 'server=<ECS Container IP address>;port=8500';
```

To get the **ECS Container IP address**, you can check the Amazon ECS tasks: <https://console.aws.amazon.com/ecs/home?#/clusters/hxe-eml-cluster/tasks>

Now that the remote source was added, you will need to reload the EML configuration as this one is loaded once at the SAP HANA, express edition startup time.

Using the **HXE** connection with the **`ML_USER`** user credentials, execute the following SQL statement:

```SQL
-- Uncomment the following lines if you want to re-run the script
-- DROP TABLE UPDATE_CONFIGURATION_PARAMS;
-- DROP TABLE UPDATE_CONFIGURATION_RESULT;
-- DROP PROCEDURE UPDATE_CONFIGURATION;

CREATE TABLE UPDATE_CONFIGURATION_PARAMS ("Parameter" VARCHAR(100), "Value" VARCHAR(100));
CREATE TABLE UPDATE_CONFIGURATION_RESULT ("Key" VARCHAR(100), "Value" INTEGER, "Text" VARCHAR(100));
CREATE PROCEDURE UPDATE_CONFIGURATION() AS
BEGIN
    DECLARE CURSOR CUR FOR
        SELECT VOLUME_ID FROM SYS.M_VOLUMES WHERE SERVICE_NAME = 'indexserver';
    FOR CUR_ROW AS CUR DO
        EXEC 'CALL _SYS_AFL.EML_CTL_PROC(''UpdateModelConfiguration'', UPDATE_CONFIGURATION_PARAMS, UPDATE_CONFIGURATION_RESULT)'
            || ' WITH OVERVIEW WITH HINT(ROUTE_TO(' || :CUR_ROW.VOLUME_ID || '))';
    END FOR;
END;
TRUNCATE TABLE UPDATE_CONFIGURATION_RESULT;
CALL UPDATE_CONFIGURATION();
SELECT * FROM UPDATE_CONFIGURATION_RESULT;
```

It should return the following result:

|    Key | Value |  Text |
|--------|-------|-------|
| Status |     0 |    OK |


> ### **Note:**: If you are using HDBSQL, you need to enable the multi-line mode using the following command in order to successfully run the above commands:
>
```sql
\mu
```

Now, you can check the registered models:

```SQL
-- Uncomment the following lines if you want to re-run the script
-- DROP TABLE CHECK_PARAMS;
CREATE TABLE CHECK_PARAMS ("Parameter" VARCHAR(100), "Value" VARCHAR(100));
TRUNCATE TABLE CHECK_PARAMS;
INSERT INTO CHECK_PARAMS VALUES ('Model', '*');
CALL _SYS_AFL.EML_CHECKDESTINATION_PROC(CHECK_PARAMS, ?);
```

Provide an answer to the question below then click on **Validate**.

[VALIDATE_1]
[ACCORDION-END]
