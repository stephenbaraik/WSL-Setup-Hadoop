# Hadoop 3.4.1 + Pig 0.17.0 on WSL (Ubuntu 22.04)

This guide sets up Hadoop and Pig **inside WSL2** on Windows, under your own user (`steve` in this example).

---

## 1️⃣ Install WSL + Ubuntu
In **PowerShell** (Admin):
```powershell
wsl --install -d Ubuntu-22.04
```
Reboot, then open Ubuntu.

---

## 2️⃣ Install Java & SSH
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install openjdk-8-jdk ssh rsync -y
sudo update-alternatives --config java
sudo update-alternatives --config javac
java -version
```
Ensure it shows `1.8.0_xxx`.

---

## 3️⃣ Download & Install Hadoop
```bash
cd ~
wget https://downloads.apache.org/hadoop/common/hadoop-3.4.1/hadoop-3.4.1.tar.gz
tar -xvzf hadoop-3.4.1.tar.gz
mv hadoop-3.4.1 hadoop
```

---

## 4️⃣ Set Environment Variables
```bash
nano ~/.bashrc
```
Append:
```bash
export HADOOP_HOME=$HOME/hadoop
export HADOOP_INSTALL=$HADOOP_HOME
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
```
Reload:
```bash
source ~/.bashrc
```

---

## 5️⃣ Configure Hadoop

### `hadoop-env.sh`
```bash
nano $HADOOP_HOME/etc/hadoop/hadoop-env.sh
```
Set:
```bash
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
```

### `core-site.xml`
```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
```

### `hdfs-site.xml`
```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:///home/steve/hadoopdata/hdfs/namenode</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:///home/steve/hadoopdata/hdfs/datanode</value>
    </property>
</configuration>
```

### `mapred-site.xml`
```bash
cp $HADOOP_HOME/etc/hadoop/mapred-site.xml.template $HADOOP_HOME/etc/hadoop/mapred-site.xml
```
```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>yarn.app.mapreduce.am.env</name>
        <value>HADOOP_MAPRED_HOME=/home/steve/hadoop</value>
    </property>
    <property>
        <name>mapreduce.map.env</name>
        <value>HADOOP_MAPRED_HOME=/home/steve/hadoop</value>
    </property>
    <property>
        <name>mapreduce.reduce.env</name>
        <value>HADOOP_MAPRED_HOME=/home/steve/hadoop</value>
    </property>
</configuration>
```

### `yarn-site.xml`
```xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
</configuration>
```

---

## 6️⃣ Setup SSH
```bash
ssh-keygen -t rsa -P ''
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 0600 ~/.ssh/authorized_keys
```

---

## 7️⃣ Format & Start Hadoop
```bash
hdfs namenode -format
start-dfs.sh
start-yarn.sh
jps
```
You should see:
```
NameNode
DataNode
ResourceManager
NodeManager
SecondaryNameNode
```

---

## 8️⃣ Install Pig
```bash
cd ~
wget https://downloads.apache.org/pig/pig-0.17.0/pig-0.17.0.tar.gz
tar -xvzf pig-0.17.0.tar.gz
mv pig-0.17.0 pig
nano ~/.bashrc
```
Add:
```bash
export PIG_HOME=$HOME/pig
export PATH=$PATH:$PIG_HOME/bin
```
Reload:
```bash
source ~/.bashrc
pig -version
```

---

## 9️⃣ Test Pig
```bash
hdfs dfs -mkdir -p /user/steve/input
echo "Hello Pig loves Hadoop" > input.txt
hdfs dfs -put input.txt /user/steve/input

nano wordcount.pig
```
```pig
fs -rm -r /user/steve/output
lines = LOAD '/user/steve/input' USING TextLoader() AS (line:chararray);
words = FOREACH lines GENERATE FLATTEN(TOKENIZE(line)) AS word;
grouped = GROUP words BY word;
wordcount = FOREACH grouped GENERATE group, COUNT(words);
STORE wordcount INTO '/user/steve/output' USING PigStorage('\t');
```
Run:
```bash
pig wordcount.pig
```
Check:
```bash
hdfs dfs -cat /user/steve/output/part-r-00000
```
Example output:
```
Pig     2
Hello   2
loves   1
Hadoop  2
```

---

✅ Hadoop 3.4.1 + Pig 0.17.0 is now fully running inside WSL.
