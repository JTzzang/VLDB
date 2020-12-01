# Running YCSB@RocksDB On Docker Container
# Thanks

https://github.com/Csoyee/documents/blob/master/KVStore/RocksDB_YCSB.md
https://github.com/Csoyee/documents/blob/master/Workload/YCSB_How_To_Start.md
https://github.com/brianfrankcooper/YCSB

Seo Sung Youn(SKKU VLDB Lab)

# Download YCSB and Compile On Local Server

# 1. Install Prerequisite
[Ref] https://github.com/Csoyee/documents/blob/master/Workload/YCSB_How_To_Start.md
  # -1) JAVA(OPEN-JDK>=8)
  ```bash
  $ sudo apt-get install openjdk-9-jdk
 ```
 
- 만일 openjdk 버전 8이상을 설치할 수 없으면 아래와 같이 한 다음에 install 
```bash
$ sudo add-apt-repository ppa:openjdk-r/ppa
$ sudo apt-get update 
```

# -2) Maven(>=3.1)
- YCSB는 Maven3 version  을 필요로 함. (Maven2를 사용할 경우 [다음](https://github.com/brianfrankcooper/YCSB/issues/406)과 같은 에러 발생)
```bash
$ sudo apt-get install maven
$ mvn --version
```
- 만일 설치된 maven 버전이 3.1 미만이면 아래와 같이 설치
```bash
$ sudo add-apt-repository ppa:andrei-pozolotin/maven3
$ sudo apt-get update
$ sudo apt-get install maven3
```

# 2. Download and Complie
``` bash
$ sudo apt-get install git 
$ git clone git clone https://github.com/brianfrankcooper/YCSB
$ cd YCSB
$ mvn clean package #매우 오래걸린다.
```
# 3. Data Load and Run(150GB Load, 50GB Write_Only  30m warm_up(초반 compaction 때문에)
```bash
# Modify Log File Path
vi YCSB/rocksdb/src/main/java/site/ycsb/db/rocksdb/RocksDBClient.java
#아래와 같이 수정
.
.
.
(생략)

 if(cfDescriptors.isEmpty()) {
151       final Options options = new Options()
152           .optimizeLevelStyleCompaction()
153           .setCreateIfMissing(true)
154           .setCreateMissingColumnFamilies(true)
155           .setIncreaseParallelism(rocksThreads)
156           .setMaxBackgroundCompactions(rocksThreads)
157           .setStatsDumpPeriodSec(60)
158           .setDbLogDir("/home/osj/workspace/log/ycsb")
159           .setInfoLogLevel(InfoLogLevel.INFO_LEVEL);
160       dbOptions = options;
161       return RocksDB.open(options, rocksDbDir.toAbsolutePath().toString());
162     } else {
163       final DBOptions options = new DBOptions()
164           .setCreateIfMissing(true)
165           .setCreateMissingColumnFamilies(true)
166           .setIncreaseParallelism(rocksThreads)
167           .setMaxBackgroundCompactions(rocksThreads)
168           .setStatsDumpPeriodSec(60)
169           .setDbLogDir("/home/osj/workspace/log/ycsb")
170           .setInfoLogLevel(InfoLogLevel.INFO_LEVEL);
171       dbOptions = options;
(생략)
.
.
.
```
## Modify Workload for Data Size
도커로 할 때에는 컨테이너 실행한 후(밑부분에 있는 sudo docker run -idt ..., sudo docker exec -it [container_name] /bin/bash으로 실행)
```bash
cd workloads
cp workloada ../[script_dir]/
exit
```
local
```bash
cd [script_dir]
cp workloada workload_r
cp workloada workload_w
vi workload_r
아래와 같이 수정
vi workload_w
아래와 같이 수정
```
이후 컨테이너 다시 실행하여 Load 과정으로 넘어감...


```bash
$ cd [Local YCSB dir]
$ mkdir script
$ cd script
$ cp -r ../workloads/ .
$ cp workloada workloada_r
$ cp workloada workload_w
```
-----아래와 같이 수정-----

-workload_r
```bash
recordcount=150000000
operationcount=50000000
workload=site.ycsb.workloads.CoreWorkload

readallfields=true

readproportion=0.95
updateproportion=0.05
scanproportion=0
insertproportion=0

requestdistribution=uniform
```
-workload_w

```bash
recordcount=150000000
operationcount=50000000
workload=site.ycsb.workloads.CoreWorkload

readallfields=true

readproportion=0
updateproportion=1.0
scanproportion=0
insertproportion=0

requestdistribution=uniform
```

```bash
# Load  
$ ./bin/ycsb load rocksdb -s -P script/workload_r -p rocksdb.dir=[data_dir] (이 때 data_dir 은 local 이 아닌 컨테이너 내 data_dir(/app/YCSB/data)로 입력
# Warmup
$ ./bin/ycsb run rocksdb -s -P script/workload_w -p maxexecutiontime=1800 -p rocksdb.dir=[data_dir]
# Backup
$ cp -r [data_dir]/* [backup_dir]/
# Run   
$ ./bin/ycsb run rocksdb -s -P script/workloada -p maxexecutiontime=[max_time] -p rocksdb.dir=[data_dir]
```

## Rocksdb configuration  
- rocksdb.dir (required): Rocksdb 데이터 파일이 위치할 디렉토리를 configuration으로 지정해준다. 


## Option 조정하기
- rocksdb 의 경우 configuration을 parameter로 조정하도록 되어있지 않기 때문에 Client에서 option을 조정하거나 parameter을 지정해주는 인터페이스는를 추가해주어야 한다. 

## Rocksdb Version 조정하기
- YCSB root 디렉토리에서 pom.xml 에 지정된 rocksdb.version 변수 수정


## Configuration 추가하기 
[참고 링크 - compaction thread 개수 option 조정](https://github.com/Csoyee/YCSB/commit/c04863a2035e763c6b6751ec0b5034db93075a40)
- rocksdb/src/main/java/com/yahoo/ycsb/db/rocksdb/RocksDBClient.java 파일에서 property 받는 코드를 추가합니다.
- 아래 예제는 `-p rocksdb.comp=(bg compaction thread 개수)` 옵션을 통해 rocksdb 백그라운드 compaction thread 개수를 조정하는 코드이다.
1. property 이름 추가하기  
    > `static final String PROPERTY_ROCKSDB_COMP_THREAD = rocksdb.comp]`
2. getProperty 함수를 이용해서 option 받기  
    > `String compThreadString = getProperties().getProperty(PROPERTY_ROCKSDB_COMP_THREAD)`  
    - 만일 받고자하는 인자가 integer 이면,   
    `import java.lang.Integer.*;` 로 인자 임포트하고   
    `int compThread = Integer.parseInt(compThreadString)` 으로 integer 값 받아옴.



# YCSB@RocksDB on Docker Container



## Create AND Run
1. Mount Device and Permission for Directory 

```bash
$ sudo mount [Device_Name] [data_dir]
$ sudo chown -R [user_name]:[user_name] [data_dir]
$ sudo chmod 777 [data_dir]
$ mkdir[script_dir]/log
```
(IF YOU AlREADY HAVE DATA_BACKUP, YOU CAN USE THAT DATA BY USING COPY&PASTE)
```bash
$ cp -r [backup_dir]/* [data_dir]
```

2. Create YCSB@RocksDB Docker Container And Run Workload
```bash
$ sudo docker run -it --rm --name ycsb \ #when process is done, remove container
  -v [data_dir]:/app/YCSB/data \
  -v [script_dir]:/app/YCSB/script \
  csoyee/ycsb:1.0 \
/bin/bash -c "./bin/ycsb run rocksdb -s \
  -P /app/YCSB/script/[workload_name] \
  -p maxexecutiontime=[execution time] \
  -p rocksdb.dir=/app/YCSB/data &> /app/YCSB/script/[log_name].log"
```


# 실험 스크립트
## [참고](https://github.com/OhSeungJin/VLDB/blob/master/tool/test_script/ycsb@rocksdb%20on%20container.md#1testsh-ycsb_only_script)
    
