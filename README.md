# Oracle 19c CDC with Debezium and Kafka

This repository demonstrates how to set up a complete Change Data Capture (CDC) pipeline for local development and testing purposes, using Oracle Database 19c, Debezium, and Apache Kafka. It captures row-level changes (Inserts, Updates, Deletes) from the database in real-time and streams them to Kafka topics.

>[!NOTE]
>A raw, sequential log of all the exact terminal and SQL commands executed to build this architecture from scratch is available in the `commands.txt` file included in this repository. You can use it as a quick reference or troubleshooting guide.

## 🚀 Prerequisites and Environment Setup

Before starting the CDC configuration, you need to have the Oracle Database 19c Docker image built locally on your machine. The `docker-compose.yml` will handle spinning up the container.

**Oracle 19c Docker Image:**
Build the Oracle image following the official Oracle guide: 
[Oracle DB 19c com Docker](https://www.oracle.com/br/technical-resources/articles/database-performance/oracle-db19c-com-docker.html). Ensure the image is tagged correctly so the `docker-compose.yml` can find it.

### Kafka Connect Configuration (Volume Mount Strategy)
To use the Debezium Oracle Connector, the Kafka Connect container must be configured with the necessary `.jar` files:

1. Download the [Debezium Oracle Connector archive](https://debezium.io/releases/3.5/).
2. Place all the extracted Debezium `.jar` files into the `./data/connect-jars/` directory in this repository. The `docker-compose.yml` will mount this folder into the Kafka Connect container's `plugin.path`.

---

## 🏗️ Step 1: Spin Up the Infrastructure

Once the Oracle image is built and the Kafka Connect plugins are in place, start the entire stack (Oracle, Kafka, and Kafka Connect):

```bash
docker compose up -d
```
>[!NOTE]
>The Oracle 19c container may take 5 to 15 minutes to fully initialize its database on the first run. Check the logs (`docker logs -f oracle`) and wait for the "DATABASE IS READY TO USE!" message before proceeding.

---

## ⚙️ Step 2: Database Configuration (ArchiveLog & Supplemental Logging)

Debezium requires the database to run in `ARCHIVELOG` mode and needs Supplemental Logging enabled to capture the "before" and "after" state of the rows.

Access the Oracle container and set up the recovery area:

```SQL
docker exec -it oracle-cdc bash
ORACLE_SID=ORACLCDB
mkdir -p /opt/oracle/oradata/recovery_area
sqlplus / as sysdba
```

Configure the recovery file destination and enable `ARCHIVELOG` mode:

```SQL
ALTER SYSTEM SET db_recovery_file_dest_size = 10G;
ALTER SYSTEM SET db_recovery_file_dest = '/opt/oracle/oradata/recovery_area' SCOPE=SPFILE;
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
```

Enable Supplemental Logging:

```SQL
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
EXIT;
```

---

## 🔐 Step 3: Creating the Debezium User

Debezium needs a dedicated user (`c##dbzuser`) with specific LogMiner privileges to read the database redo logs.

Create tablespaces for LogMiner in both the Container Database (CDB) and Pluggable Database (PDB):

```bash
# In CDB
sqlplus sys/password@//localhost:1521/ORCLCDB as sysdba
CREATE TABLESPACE logminer_tbs DATAFILE '/opt/oracle/oradata/ORCLCDB/logminer_tbs.dbf' SIZE 25M REUSE AUTOEXTEND ON MAXSIZE UNLIMITED;
EXIT;

# In PDB
sqlplus sys/password@//localhost:1521/ORCLPDB1 as sysdba
CREATE TABLESPACE logminer_tbs DATAFILE '/opt/oracle/oradata/ORCLCDB/ORCLPDB1/logminer_tbs.dbf' SIZE 25M REUSE AUTOEXTEND ON MAXSIZE UNLIMITED;
EXIT;
```

Create the user and grant all necessary privileges for LogMining:

```bash
sqlplus sys/password@//localhost:1521/ORCLCDB as sysdba
```

```SQL
CREATE USER c##dbzuser IDENTIFIED BY dbz DEFAULT TABLESPACE logminer_tbs QUOTA UNLIMITED ON logminer_tbs CONTAINER=ALL;

GRANT CREATE SESSION TO c##dbzuser CONTAINER=ALL;
GRANT SET CONTAINER TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ON V_$DATABASE to c##dbzuser CONTAINER=ALL;
GRANT FLASHBACK ANY TABLE TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ANY TABLE TO c##dbzuser CONTAINER=ALL;
GRANT SELECT_CATALOG_ROLE TO c##dbzuser CONTAINER=ALL;
GRANT EXECUTE_CATALOG_ROLE TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ANY TRANSACTION TO c##dbzuser CONTAINER=ALL;
GRANT LOGMINING TO c##dbzuser CONTAINER=ALL;
GRANT ALTER SESSION, SET CONTAINER TO c##dbzuser CONTAINER=ALL;
GRANT CREATE TABLE TO c##dbzuser CONTAINER=ALL;
GRANT LOCK ANY TABLE TO c##dbzuser CONTAINER=ALL;
GRANT CREATE SEQUENCE TO c##dbzuser CONTAINER=ALL;
GRANT EXECUTE ON DBMS_LOGMNR TO c##dbzuser CONTAINER=ALL; 
GRANT EXECUTE ON DBMS_LOGMNR_D TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ON V_$LOG TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ON V_$LOG_HISTORY TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ON V_$LOGMNR_LOGS TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ON V_$LOGMNR_CONTENTS TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ON V_$LOGMNR_PARAMETERS TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ON V_$LOGFILE TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ON V_$ARCHIVED_LOG TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ON V_$ARCHIVE_DEST_STATUS TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ON V_$ARCHIVE_DEST_STATUS TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ON V_$TRANSACTION TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ON V_$MYSTAT TO c##dbzuser CONTAINER=ALL;
GRANT SELECT ON V_$STATNAME TO c##dbzuser CONTAINER=ALL;
exit;
```

---

## 📊 Step 4: Application Schema & Test Data Generation

Create an application user and a table to monitor, then generate test data to trigger CDC events.

```bash
sqlplus sys/password@//localhost:1521/ORCLPDB1 as sysdba
```

```SQL
CREATE USER appuser IDENTIFIED BY apppassword;
GRANT CONNECT, RESOURCE TO appuser;
ALTER USER appuser QUOTA UNLIMITED ON users;

CREATE TABLE appuser.test_cdc (id NUMBER PRIMARY KEY, status VARCHAR2(50));
```

Run this PL/SQL block to simulate a real-world workload (100 inserts, 50 updates, and 20 deletes):

```SQL
BEGIN
    -- 1. Creates new records (Testing "op": "c")
    FOR i IN 1..100 LOOP
        INSERT INTO appuser.test_cdc (id, status) VALUES (i, 'RECEIVED');
    END LOOP;
    COMMIT;

    -- 2. Updates 50 records (Testing "op": "u")
    FOR i IN 1..50 LOOP
        UPDATE appuser.test_cdc SET status = 'PROCESSING' WHERE id = i;
    END LOOP;
    COMMIT;

    -- 3. Deletes 20 records (Testing "op": "d")
    FOR i IN 81..100 LOOP
        DELETE FROM appuser.test_cdc WHERE id = i;
    END LOOP;
    COMMIT;
END;
/
EXIT;
```

---

## 🔌 Step 5: Deploying the Debezium Connector

Verify that Kafka Connect has loaded the Oracle connector successfully:

```bash
curl -s localhost:8083/connector-plugins | jq
```

Deploy the connector by submitting your JSON configuration file:

```bash
curl -i -X POST -H "Content-Type:application/json" http://localhost:8083/connectors -d @oracle-connector.json
```

Check the status of the connector to ensure it is running without errors:

```bash
curl -s localhost:8083/connectors/oracle-connector/status | jq
```

>[!NOTE]
>If you need to debug, run `docker logs -f kafka-connect`

---

## 📡 Step 6: Verifying CDC Events in Kafka

Once the connector is running, it will read the database history and stream the changes. To view the generated JSON payloads in real-time, consume the Kafka topic directly from the Kafka container:

``` bash
docker exec -i kafka /opt/kafka/bin/kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic cdc_teste.APPUSER.TEST_CDC \
--from-beginning | jq .
```

You will see JSON messages representing the database operations:

- `"op": "r"`: (Read - Initial Snapshot)
- `"op": "c"`: (Create - Inserts)
- `"op": "u"`: (Update)
- `"op": "d"`: (Delete)
