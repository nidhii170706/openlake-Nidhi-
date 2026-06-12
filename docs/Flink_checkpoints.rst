
===================
 OpenLake Documentation Structure
===================
.. contents:: On this page
   :depth: 2

# Overview

This example shows how Apache Flink can read
and write data to OpenLake storage.

What is Flink?
--------------

Apache Flink is a real-time stream processing
framework. It can process large amounts of data
very fast and store results in OpenLake.

Prerequisites
-------------

Before starting, make sure you have:

- OpenLake cluster running
- Apache Flink installed
- Java 11 or higher
- S3 credentials configured

Setting Up OpenLake Cluster
----------------------------

Step 1: Start OpenLake
~~~~~~~~~~~~~~~~~~~~~~

Start your OpenLake node:

.. code-block:: bash

   ./openlaked --config node.toml

Expected output from your terminal:

.. code-block:: bash

   [INFO] OpenLake v1.0.0 starting...
   [INFO] Storage engine initialized
   [INFO] Server listening on :9000
   [INFO] S3 API listening on :9001
   [INFO] Ready to serve requests! ✅

Step 2: Create a Bucket
~~~~~~~~~~~~~~~~~~~~~~~

Create a bucket where Flink will store data:

.. code-block:: bash

   ./openlake bucket create flink-data

Expected output:

.. code-block:: bash

   Bucket 'flink-data' created successfully ✅

Step 3: Verify Cluster Health
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   curl http://localhost:9000/health

Expected output:

.. code-block:: bash

   {"status": "healthy", "version": "1.0.0"} ✅

Setting Up Flink
----------------

Step 1: Download Flink
~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   wget https://flink.apache.org/downloads/
        flink-1.18.0-bin-scala_2.12.tgz
        
   tar xzf flink-1.18.0-bin-scala_2.12.tgz
   cd flink-1.18.0

Step 2: Configure Flink for OpenLake
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Add these settings to conf/flink-conf.yaml:

.. code-block:: yaml

   # OpenLake S3 connection settings
   s3.endpoint: http://localhost:9001
   s3.access-key: admin
   s3.secret-key: password123
   s3.path.style.access: true

Step 3: Start Flink Cluster
~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   ./bin/start-cluster.sh

Expected output:

.. code-block:: bash

   Starting cluster.
   Starting standalonesession daemon on host.
   Starting taskexecutor daemon on host.
   Flink is now running! ✅

Reading Data from OpenLake
---------------------------

This example reads a CSV file stored in OpenLake:

.. code-block:: python

   # Python Flink Example - Read from OpenLake
   
   from pyflink.datastream import \
       StreamExecutionEnvironment

   # Create environment
   env = StreamExecutionEnvironment\
         .get_execution_environment()

   # Read from OpenLake using S3 path
   data_stream = env.read_text_file(
       "s3://flink-data/input/data.csv"
   )

   # Print the data
   data_stream.print()

   # Run the job
   env.execute("Read from OpenLake")

Expected terminal output:

.. code-block:: bash

   $ python read_example.py
   
   Job ID: abc123def456
   Reading from: s3://flink-data/input/data.csv
   
   name,age,city
   John,25,Mumbai
   Alice,30,Delhi
   Bob,28,Bangalore
   
   Job completed successfully! ✅

Writing Data to OpenLake
-------------------------

This example processes data and writes to OpenLake:

.. code-block:: python

   # Python Flink Example - Write to OpenLake
   
   from pyflink.datastream import \
       StreamExecutionEnvironment

   env = StreamExecutionEnvironment\
         .get_execution_environment()

   # Create sample data stream
   data_stream = env.from_collection([
       "John,25,Mumbai",
       "Alice,30,Delhi",
       "Bob,28,Bangalore"
   ])

   # Write to OpenLake
   data_stream.write_as_text(
       "s3://flink-data/output/results.txt"
   )

   env.execute("Write to OpenLake")

Expected terminal output:

.. code-block:: bash

   $ python write_example.py
   
   Job ID: xyz789abc123
   Writing to: s3://flink-data/output/results.txt
   Records written: 3
   
   Job completed successfully! ✅
   
   # Verify in OpenLake:
   $ ./openlake object list \
       --bucket flink-data \
       --prefix output/
   
   output/results.txt   124 bytes   ✅

Flink Checkpoints to OpenLake
------------------------------

Checkpoints = Flink saves its progress
If job fails, it can restart from checkpoint
OpenLake can store these checkpoints safely:

.. code-block:: python

   from pyflink.datastream import \
       StreamExecutionEnvironment
   from pyflink.datastream.checkpoint_config import \
       CheckpointingMode

   env = StreamExecutionEnvironment\
         .get_execution_environment()

   # Save checkpoints to OpenLake
   env.get_checkpoint_config()\
      .set_checkpoint_storage(
          "s3://flink-data/checkpoints/"
      )

   # Enable checkpointing every 60 seconds
   env.enable_checkpointing(60000)
   
   # Set checkpointing mode
   env.get_checkpoint_config()\
      .set_checkpointing_mode(
          CheckpointingMode.EXACTLY_ONCE
      )

   # Your processing logic here
   data_stream = env.from_collection([
       "record1", "record2", "record3"
   ])
   data_stream.print()

   env.execute("Flink Checkpoint Example")

Expected output:

.. code-block:: bash

   $ python checkpoint_example.py
   
   Checkpointing enabled: every 60 seconds
   Checkpoint location: s3://flink-data/checkpoints/
   
   Processing records...
   Checkpoint 1 saved successfully ✅
   Checkpoint 2 saved successfully ✅
   
   # Check checkpoints in OpenLake:
   $ ./openlake object list \
       --bucket flink-data \
       --prefix checkpoints/
   
   checkpoints/chk-1/   45 KB   ✅
   checkpoints/chk-2/   45 KB   ✅

Complete Working Example
------------------------

Full end-to-end example running everything together:

.. code-block:: bash

   # ── STEP 1: Start OpenLake ──
   ./openlaked --config node.toml
   # [INFO] Ready to serve requests! ✅


   # ── STEP 2: Create bucket ──
   ./openlake bucket create flink-data
   # Bucket created ✅


   # ── STEP 3: Upload input data ──
   cat > data.csv << EOF
   name,age,city
   John,25,Mumbai
   Alice,30,Delhi
   Bob,28,Bangalore
   EOF

   ./openlake object put \
       --bucket flink-data \
       --key input/data.csv \
       --file data.csv
   # Upload successful ✅


   # ── STEP 4: Start Flink ──
   cd flink-1.18.0
   ./bin/start-cluster.sh
   # Flink running ✅


   # ── STEP 5: Run Flink job ──
   python flink_openlake_example.py
   # Job completed ✅


   # ── STEP 6: Check output ──
   ./openlake object get \
       --bucket flink-data \
       --key output/results.txt \
       --output results.txt

   cat results.txt
   # name,age,city
   # John,25,Mumbai
   # Alice,30,Delhi
   # Bob,28,Bangalore ✅

