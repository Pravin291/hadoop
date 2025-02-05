

```markdown
## Multi-node Hadoop Cluster Installation Guide (Ubuntu)

This guide provides step-by-step instructions to set up a **multi-node Hadoop cluster** on **Ubuntu**. It includes the installation of Hadoop, configuration of master and worker nodes, and setting up necessary services.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Install SSH and Configure Passwordless SSH](#install-ssh-and-configure-passwordless-ssh)
3. [Create a Hadoop User](#create-a-hadoop-user)
4. [Install Java](#install-java)
5. [Set Up Environment Variables](#set-up-environment-variables)
6. [Set Up Passwordless SSH](#set-up-passwordless-ssh)
7. [Download Hadoop](#download-hadoop)
8. [Configure Hadoop](#configure-hadoop)
9. [Start Hadoop Daemons](#start-hadoop-daemons)
10. [Check Cluster Status](#check-cluster-status)

---

## 1. Prerequisites

Ensure the following prerequisites are met before proceeding:

- Ubuntu system with root or sudo access.
- At least **3 nodes**: 1 master and 2 worker nodes.

### Update System

```bash
sudo apt update
```

---

## 2. Install SSH and Configure Passwordless SSH

1. **Install SSH:**

   ```bash
   sudo apt install ssh
   sudo systemctl enable ssh
   sudo systemctl start ssh
   ```

2. **Create Hadoop User (`hduser`):**

   ```bash
   sudo adduser hduser
   ```

3. **Grant Sudo Permissions to `hduser`:**

   Open the sudoers file:

   ```bash
   sudo visudo
   ```

   Add the following line:

   ```bash
   hduser ALL=(ALL:ALL) NOPASSWD:ALL
   ```

   Save and exit the file.

---

## 3. Install Java

1. **Install OpenJDK 8:**

   ```bash
   sudo apt install openjdk-8-jdk
   ```

2. **Confirm Java Installation:**

   ```bash
   java -version
   ```

---

## 4. Set Up Environment Variables

1. **Edit the `.bashrc` File for `hduser`:**

   Add the following lines to set environment variables:

   ```bash
   # JAVA
   export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/
   export JRE_HOME=/usr/lib/jvm/java-8-openjdk-amd64/jre

   # Hadoop Environment Variables
   export HADOOP_HOME=/usr/local/hadoop
   export HADOOP_CONF_DIR=/usr/local/hadoop/etc/hadoop
   export HADOOP_LOG_DIR=$HADOOP_HOME/logs
   export HADOOP_MAPRED_HOME=$HADOOP_HOME
   export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
   export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
   export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib/native"
   export PDSH_RCMD_TYPE=ssh
   ```

2. **Reload the `.bashrc` File:**

   ```bash
   source ~/.bashrc
   ```

---

## 5. Set Up Passwordless SSH

1. **Generate SSH Key Pair:**

   ```bash
   ssh-keygen -t rsa
   ```

   This will create an `.ssh` folder with `id_rsa` and `id_rsa.pub` files.

2. **Copy the Public Key to Master and Worker Nodes:**

   ```bash
   ssh-copy-id -i ~/.ssh/id_rsa.pub hduser@localhost
   ssh-copy-id -i ~/.ssh/id_rsa.pub hduser@node1
   ssh-copy-id -i ~/.ssh/id_rsa.pub hduser@node2
   ```

3. **Test SSH Access:**

   Ensure you can SSH into the nodes without a password prompt:

   ```bash
   ssh hduser@localhost
   ssh hduser@node1
   ssh hduser@node2
   ```

---

## 6. Download Hadoop

1. **Download Hadoop:**

   ```bash
   wget -c -O hadoop.tar.gz https://dlcdn.apache.org/hadoop/common/hadoop-3.2.4/hadoop-3.2.4.tar.gz
   ```

2. **Extract Hadoop:**

   ```bash
   sudo mkdir /usr/local/hadoop
   tar xvzf hadoop.tar.gz
   sudo mv hadoop-3.2.4/* /usr/local/hadoop
   ```

---

## 7. Configure Hadoop

1. **Create Directories for NameNode and DataNode:**

   ```bash
   mkdir -p /usr/local/hadoop/hd_store/tmp
   mkdir -p /usr/local/hadoop/hd_store/namenode
   mkdir -p /usr/local/hadoop/hd_store/datanode
   ```

2. **Change Ownership and Permissions:**

   ```bash
   sudo chown -R hduser:hduser /usr/local/hadoop
   sudo chmod 755 -R /usr/local/hadoop
   ```

3. **Configure Hadoop Files:**

   - **`hadoop-env.sh`**: Set the `JAVA_HOME` and other Hadoop environment variables.
   
     Edit `/usr/local/hadoop/etc/hadoop/hadoop-env.sh`:
   
     ```bash
     export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/
     export JRE_HOME=/usr/lib/jvm/java-8-openjdk-amd64/jre
     export HADOOP_HOME=/usr/local/hadoop
     export HADOOP_CONF_DIR=/usr/local/hadoop/etc/hadoop
     export HADOOP_LOG_DIR=$HADOOP_HOME/logs
     export HDFS_NAMENODE_USER=hduser
     export HDFS_DATANODE_USER=hduser
     export HDFS_SECONDARYNAMENODE_USER=hduser
     export YARN_RESOURCEMANAGER_USER=hduser
     export YARN_NODEMANAGER_USER=hduser
     export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
     export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
     export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib/native"
     ```

   - **`core-site.xml`**: Set the default filesystem.
   
     Edit `/usr/local/hadoop/etc/hadoop/core-site.xml`:
   
     ```xml
     <configuration>
         <property>
             <name>fs.defaultFS</name>
             <value>hdfs://master:9000</value>
         </property>
     </configuration>
     ```

   - **`yarn-site.xml`**: Set the ResourceManager hostname.
   
     Edit `/usr/local/hadoop/etc/hadoop/yarn-site.xml`:
   
     ```xml
     <configuration>
         <property>
             <name>yarn.resourcemanager.hostname</name>
             <value>master</value>
         </property>
     </configuration>
     ```

   - **`hdfs-site.xml`**: Configure NameNode and DataNode directories.
   
     Edit `/usr/local/hadoop/etc/hadoop/hdfs-site.xml`:
   
     ```xml
     <configuration>
         <property>
             <name>dfs.name.dir</name>
             <value>/usr/local/hadoop/hd-store/namenode</value>
         </property>
         <property>
             <name>dfs.replication</name>
             <value>2</value>
         </property>
     </configuration>
     ```

   - **`mapred-site.xml`**: Configure MapReduce framework.
   
     Edit `/usr/local/hadoop/etc/hadoop/mapred-site.xml`:
   
     ```xml
     <configuration>
         <property>
             <name>mapreduce.framework.name</name>
             <value>yarn</value>
         </property>
         <property>
             <name>mapreduce.application.classpath</name>
             <value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
         </property>
     </configuration>
     ```

4. **Configure Worker Nodes:**

   Add worker nodes to the **`workers`** file:
   
   ```bash
   node1
   node2
   ```

---

## 8. Start Hadoop Daemons

1. **Format the NameNode:**

   ```bash
   hadoop namenode -format
   ```

2. **Start HDFS and YARN Services:**

   ```bash
   start-dfs.sh
   start-yarn.sh
   ```

---

## 9. Check Cluster Status

Use the `jps` command to verify the daemons are running:

```bash
jps
```

You should see:
- NameNode
- DataNode
- ResourceManager
- NodeManager

Access the Hadoop UI:

- **HDFS Web UI:**  
  [http://<master-ip>:50070/](http://<master-ip>:50070/)
  
- **YARN Web UI:**  
  [http://<master-ip>:8088/](http://<master-ip>:8088/)

---

## Your Multinode hadoop cluster is ready for use!!!.
