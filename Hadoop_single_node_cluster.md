```markdown
# Setting Up a Single Node Hadoop Cluster on Ubuntu

This guide will walk you through setting up a single-node Hadoop cluster on Ubuntu.

## 1. Prepare the Machine and Update It
First, ensure that your machine is up-to-date:

```bash
sudo apt update
```

## 2. Create a User for Hadoop
Create a new user called `hduser`:

```bash
sudo adduser hduser
```

## 3. Install SSH
To enable communication between the nodes, install SSH and start the service:

```bash
sudo apt install ssh
sudo systemctl enable ssh
sudo systemctl start ssh
```

### Provide Sudo Permission to `hduser`
Edit the sudoers file:

```bash
sudo visudo
```

Add the following line:

```bash
hduser  ALL=(ALL:ALL) NOPASSWD:ALL
```

Log in as `hduser`:

```bash
su - hduser
```

## 4. Install Java (Java 8 or Java 11)
Hadoop supports Java 8 or Java 11. Here, we will install Java 8:

```bash
sudo apt install openjdk-8-jdk
```

Check the Java installation:

```bash
java -version
```

## 5. Set Environment Variables
Edit the `~/.bashrc` file to set the necessary environment variables:

```bash
nano ~/.bashrc
```

Add the following lines:

```bash
# JAVA
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/
export JRE_HOME=/usr/lib/jvm/java-8-openjdk-amd64/jre

# Hadoop Environment Variables
export HADOOP_HOME=/usr/local/hadoop
export HADOOP_CONF_DIR=/usr/local/hadoop/etc/hadoop
export HADOOP_LOG_DIR=$HADOOP_HOME/logs
export HADOOP_MAPRED_HOME=$HADOOP_HOME

# Add Hadoop bin/ directory to PATH
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib/native"
export PDSH_RCMD_TYPE=ssh
```

Reload the `.bashrc` file:

```bash
source ~/.bashrc
```

## 6. Enable Passwordless SSH
Generate an SSH key pair:

```bash
ssh-keygen -t rsa
```

This creates a `.ssh` folder with the `id_rsa` and `id_rsa.pub` files.

Copy the public key to the local machine:

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub hduser@localhost
```

Test SSH access:

```bash
ssh hduser@localhost
```

You should not be prompted for a password.

## 7. Download Hadoop
Navigate to your home directory and download Hadoop:

```bash
cd
wget -c -O hadoop.tar.gz https://dlcdn.apache.org/hadoop/common/hadoop-3.2.4/hadoop-3.2.4.tar.gz
```

Create the directory for Hadoop and extract the tarball:

```bash
sudo mkdir /usr/local/hadoop
tar xvzf hadoop.tar.gz
sudo mv hadoop-3.2.4/* /usr/local/hadoop
```

## 8. Configure Hadoop

### Create Directories for Namenode and Datanode
Create the necessary directories for Hadoop's namenode and datanode:

```bash
mkdir -p /usr/local/hadoop/hd_store/tmp
mkdir -p /usr/local/hadoop/hd_store/namenode
mkdir -p /usr/local/hadoop/hd_store/datanode
```

Set permissions for the Hadoop directories:

```bash
sudo chown -R hduser:hduser /usr/local/hadoop
sudo chmod 755 -R /usr/local/hadoop
```

### Edit the `hadoop-env.sh` File
Edit the `hadoop-env.sh` file:

```bash
nano /usr/local/hadoop/etc/hadoop/hadoop-env.sh
```

Set the following environment variables:

```bash
# JAVA
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/
export JRE_HOME=/usr/lib/jvm/java-8-openjdk-amd64/jre

# Hadoop Environment Variables
export HADOOP_HOME=/usr/local/hadoop
export HADOOP_CONF_DIR=/usr/local/hadoop/etc/hadoop
export HADOOP_LOG_DIR=$HADOOP_HOME/logs
export HDFS_NAMENODE_USER=hduser
export HDFS_DATANODE_USER=hduser
export HDFS_SECONDARYNAMENODE_USER=hduser
export YARN_RESOURCEMANAGER_USER=hduser
export YARN_NODEMANAGER_USER=hduser

# Add Hadoop bin/ directory to PATH
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib/native"
```

### Edit the Configuration Files

#### `core-site.xml`

Edit the `core-site.xml` file:

```bash
nano /usr/local/hadoop/etc/hadoop/core-site.xml
```

Add the following configuration:

```xml
<configuration>
  <property>
    <name>hadoop.tmp.dir</name>
    <value>/usr/local/hadoop/hd_store/tmp</value>
  </property>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://master:9000</value>
  </property>
</configuration>
```

#### `yarn-site.xml`

Edit the `yarn-site.xml` file:

```bash
nano /usr/local/hadoop/etc/hadoop/yarn-site.xml
```

Add the following configuration:

```xml
<configuration>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
</configuration>
```

#### `hdfs-site.xml`

Edit the `hdfs-site.xml` file:

```bash
nano /usr/local/hadoop/etc/hadoop/hdfs-site.xml
```

Add the following configuration:

```xml
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>1</value>
  </property>
  <property>
    <name>dfs.name.dir</name>
    <value>/usr/local/hadoop/hd_store/namenode</value>
  </property>
  <property>
    <name>dfs.data.dir</name>
    <value>/usr/local/hadoop/hd_store/datanode</value>
  </property>
</configuration>
```

#### `mapred-site.xml`

Edit the `mapred-site.xml` file:

```bash
nano /usr/local/hadoop/etc/hadoop/mapred-site.xml
```

Add the following configuration:

```xml
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
  <property>
    <name>yarn.app.mapreduce.am.env</name>
    <value>HADOOP_MAPRED_HOME=$HADOOP_HOME</value>
  </property>
  <property>
    <name>mapreduce.map.env</name>
    <value>HADOOP_MAPRED_HOME=$HADOOP_HOME</value>
  </property>
  <property>
    <name>mapreduce.reduce.env</name>
    <value>HADOOP_MAPRED_HOME=$HADOOP_HOME</value>
  </property>
</configuration>
```

#### Edit the `workers` File
Edit the `workers` file:

```bash
nano /usr/local/hadoop/etc/hadoop/workers
```

Add the following entry:

```bash
localhost
```

## 9. Start Hadoop Daemons

### Format the Namenode
Run the following command to format the Namenode:

```bash
hadoop namenode -format
```

### Start the Hadoop Daemons

Start the Hadoop DFS daemons:

```bash
start-dfs.sh
```

Start the YARN daemons:

```bash
start-yarn.sh
```

### Verify the Daemons Are Running

Check the status of the Hadoop daemons using the `jps` command:

```bash
jps
```

You should see the following daemons running:

```markdown
# Setting Up a Single Node Hadoop Cluster on Ubuntu

This guide will walk you through setting up a single-node Hadoop cluster on Ubuntu.

## 1. Prepare the Machine and Update It
First, ensure that your machine is up-to-date:

```bash
sudo apt update
```

## 2. Create a User for Hadoop
Create a new user called `hduser`:

```bash
sudo adduser hduser
```

## 3. Install SSH
To enable communication between the nodes, install SSH and start the service:

```bash
sudo apt install ssh
sudo systemctl enable ssh
sudo systemctl start ssh
```

### Provide Sudo Permission to `hduser`
Edit the sudoers file:

```bash
sudo visudo
```

Add the following line:

```bash
hduser  ALL=(ALL:ALL) NOPASSWD:ALL
```

Log in as `hduser`:

```bash
su - hduser
```

## 4. Install Java (Java 8 or Java 11)
Hadoop supports Java 8 or Java 11. Here, we will install Java 8:

```bash
sudo apt install openjdk-8-jdk
```

Check the Java installation:

```bash
java -version
```

## 5. Set Environment Variables
Edit the `~/.bashrc` file to set the necessary environment variables:

```bash
nano ~/.bashrc
```

Add the following lines:

```bash
# JAVA
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/
export JRE_HOME=/usr/lib/jvm/java-8-openjdk-amd64/jre

# Hadoop Environment Variables
export HADOOP_HOME=/usr/local/hadoop
export HADOOP_CONF_DIR=/usr/local/hadoop/etc/hadoop
export HADOOP_LOG_DIR=$HADOOP_HOME/logs
export HADOOP_MAPRED_HOME=$HADOOP_HOME

# Add Hadoop bin/ directory to PATH
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib/native"
export PDSH_RCMD_TYPE=ssh
```

Reload the `.bashrc` file:

```bash
source ~/.bashrc
```

## 6. Enable Passwordless SSH
Generate an SSH key pair:

```bash
ssh-keygen -t rsa
```

This creates a `.ssh` folder with the `id_rsa` and `id_rsa.pub` files.

Copy the public key to the local machine:

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub hduser@localhost
```

Test SSH access:

```bash
ssh hduser@localhost
```

You should not be prompted for a password.

## 7. Download Hadoop
Navigate to your home directory and download Hadoop:

```bash
cd
wget -c -O hadoop.tar.gz https://dlcdn.apache.org/hadoop/common/hadoop-3.2.4/hadoop-3.2.4.tar.gz
```

Create the directory for Hadoop and extract the tarball:

```bash
sudo mkdir /usr/local/hadoop
tar xvzf hadoop.tar.gz
sudo mv hadoop-3.2.4/* /usr/local/hadoop
```

## 8. Configure Hadoop

### Create Directories for Namenode and Datanode
Create the necessary directories for Hadoop's namenode and datanode:

```bash
mkdir -p /usr/local/hadoop/hd_store/tmp
mkdir -p /usr/local/hadoop/hd_store/namenode
mkdir -p /usr/local/hadoop/hd_store/datanode
```

Set permissions for the Hadoop directories:

```bash
sudo chown -R hduser:hduser /usr/local/hadoop
sudo chmod 755 -R /usr/local/hadoop
```

### Edit the `hadoop-env.sh` File
Edit the `hadoop-env.sh` file:

```bash
nano /usr/local/hadoop/etc/hadoop/hadoop-env.sh
```

Set the following environment variables:

```bash
# JAVA
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/
export JRE_HOME=/usr/lib/jvm/java-8-openjdk-amd64/jre

# Hadoop Environment Variables
export HADOOP_HOME=/usr/local/hadoop
export HADOOP_CONF_DIR=/usr/local/hadoop/etc/hadoop
export HADOOP_LOG_DIR=$HADOOP_HOME/logs
export HDFS_NAMENODE_USER=hduser
export HDFS_DATANODE_USER=hduser
export HDFS_SECONDARYNAMENODE_USER=hduser
export YARN_RESOURCEMANAGER_USER=hduser
export YARN_NODEMANAGER_USER=hduser

# Add Hadoop bin/ directory to PATH
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib/native"
```

### Edit the Configuration Files

#### `core-site.xml`

Edit the `core-site.xml` file:

```bash
nano /usr/local/hadoop/etc/hadoop/core-site.xml
```

Add the following configuration:

```xml
<configuration>
  <property>
    <name>hadoop.tmp.dir</name>
    <value>/usr/local/hadoop/hd_store/tmp</value>
  </property>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://master:9000</value>
  </property>
</configuration>
```

#### `yarn-site.xml`

Edit the `yarn-site.xml` file:

```bash
nano /usr/local/hadoop/etc/hadoop/yarn-site.xml
```

Add the following configuration:

```xml
<configuration>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
</configuration>
```

#### `hdfs-site.xml`

Edit the `hdfs-site.xml` file:

```bash
nano /usr/local/hadoop/etc/hadoop/hdfs-site.xml
```

Add the following configuration:

```xml
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>1</value>
  </property>
  <property>
    <name>dfs.name.dir</name>
    <value>/usr/local/hadoop/hd_store/namenode</value>
  </property>
  <property>
    <name>dfs.data.dir</name>
    <value>/usr/local/hadoop/hd_store/datanode</value>
  </property>
</configuration>
```

#### `mapred-site.xml`

Edit the `mapred-site.xml` file:

```bash
nano /usr/local/hadoop/etc/hadoop/mapred-site.xml
```

Add the following configuration:

```xml
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
  <property>
    <name>yarn.app.mapreduce.am.env</name>
    <value>HADOOP_MAPRED_HOME=$HADOOP_HOME</value>
  </property>
  <property>
    <name>mapreduce.map.env</name>
    <value>HADOOP_MAPRED_HOME=$HADOOP_HOME</value>
  </property>
  <property>
    <name>mapreduce.reduce.env</name>
    <value>HADOOP_MAPRED_HOME=$HADOOP_HOME</value>
  </property>
</configuration>
```

#### Edit the `workers` File
Edit the `workers` file:

```bash
nano /usr/local/hadoop/etc/hadoop/workers
```

Add the following entry:

```bash
localhost
```

## 9. Start Hadoop Daemons

### Format the Namenode
Run the following command to format the Namenode:

```bash
hadoop namenode -format
```

### Start the Hadoop Daemons

Start the Hadoop DFS daemons:

```bash
start-dfs.sh
```

Start the YARN daemons:

```bash
start-yarn.sh
```

### Verify the Daemons Are Running

Check the status of the Hadoop daemons using the `jps` command:

```bash
jps
```

You should see the following daemons running:

- `NameNode`
- `DataNode`
- `ResourceManager`
- `NodeManager`
- `SecondaryNameNode`

If all daemons are running, your single-node Hadoop cluster is ready to use!

```
