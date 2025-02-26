# Web Server Log Analysis Using Apache Hive

## Project Overview
This project analyzes web server logs using Apache Hive to extract meaningful insights. The dataset consists of log entries in CSV format, including fields such as IP address, timestamp, requested URL, HTTP status code, and user agent. The primary objectives of the project are:
- Counting total web requests.
- Analyzing the distribution of HTTP status codes.
- Identifying the top three most visited pages.
- Evaluating the most common traffic sources (user agents).
- Detecting suspicious IPs with multiple failed requests.
- Analyzing traffic trends over time.
- Implementing partitioning by status code to optimize query performance.

## Implementation Approach

### 1. Setting Up Hive Environment
- A Hive external table is created to store the web logs.
- Data is loaded into the table from HDFS.

### 2. Queries Used for Analysis
#### Counting Total Web Requests
```sql
SELECT COUNT(*) AS total_requests FROM web_logs;
```

#### Analyzing Status Codes
```sql
SELECT status, COUNT(*) AS count
FROM web_logs
GROUP BY status;
```

#### Identifying Most Visited Pages
```sql
SELECT url, COUNT(*) AS visits
FROM web_logs
GROUP BY url
ORDER BY visits DESC
LIMIT 3;
```

#### Traffic Source Analysis
```sql
SELECT user_agent, COUNT(*) AS count
FROM web_logs
GROUP BY user_agent
ORDER BY count DESC;
```

#### Detecting Suspicious Activity (IPs with >3 failed requests)
```sql
SELECT ip, COUNT(*) AS failed_requests
FROM web_logs
WHERE status IN (404, 500)
GROUP BY ip
HAVING COUNT(*) > 3;
```

#### Analyzing Traffic Trends Over Time
```sql
SELECT substr(timestamp, 0, 16) AS minute, COUNT(*) AS requests
FROM web_logs
GROUP BY substr(timestamp, 0, 16)
ORDER BY minute;
```

### 3. Implementing Partitioning
To optimize query performance, we partition the table by status code:
```sql
CREATE TABLE web_logs_partitioned (
    ip STRING,
    timestamp STRING,
    url STRING,
    user_agent STRING
)
PARTITIONED BY (status INT)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ',';
```
Data is then inserted into the partitioned table.

## Execution Steps
1. Setup Docker and HDFS:
```bash
docker exec -it namenode /bin/bash
hdfs dfs -mkdir -p /data/web_logs
exit
```
2. Copy and Load Data:
```bash
docker cp web_server_logs.csv namenode:/data/web_logs/web_server_logs.csv
docker exec -it namenode /bin/bash
hdfs dfs -put /data/web_logs/web_server_logs.csv /data/web_logs/
exit
```
3.Start Hive and Create Table:
```bash
docker exec -it hive-server /bin/bash
hive
```
4. Run Queries from HQL File:
```bash
touch hql_queries.hql
code hql_queries.hql
docker cp hql_queries.hql hive-server:/opt/hql_queries.hql
docker exec -it hive-server /bin/bash
hive -f /opt/hql_queries.hql | tee /opt/hql_output.txt
docker cp hive-server:/opt/hql_output.txt hql_output.txt
```

## Challenges Faced
- Data Formatting Issues: Some log entries had inconsistencies in delimiters and required preprocessing.
-Partitioning Strategy:Finding the right partitioning column was crucial for optimizing queries.
- HDFS File Permissions:Ensuring proper file permissions for Hive to access the data.

## Sample Input and Expected Output
### Sample Input (CSV Format)
```
ip,timestamp,url,status,user_agent
192.168.1.1,2024-02-01 10:15:00,/home,200,Mozilla/5.0
192.168.1.2,2024-02-01 10:16:00,/products,200,Chrome/90.0
192.168.1.3,2024-02-01 10:17:00,/checkout,404,Safari/13.1
192.168.1.10,2024-02-01 10:18:00,/home,500,Mozilla/5.0
192.168.1.15,2024-02-01 10:19:00,/products,404,Chrome/90.0
```

### Expected Output
#### Total Web Requests:
```
Total Requests: 100
```
#### Status Code Analysis:
```
200: 80
404: 10
500: 10
```
#### Most Visited Pages:
```
/home: 50
/products: 30
/checkout: 20
```
#### Traffic Source Analysis:
```
Mozilla/5.0: 60
Chrome/90.0: 30
Safari/13.1: 10
```
#### Suspicious IP Addresses:
```
192.168.1.10: 5 failed requests
192.168.1.15: 4 failed requests
```
#### Traffic Trend Over Time:
```
2024-02-01 10:15: 5 requests
2024-02-01 10:16: 7 requests
```

## Conclusion
This project successfully demonstrates how Apache Hive can be used to analyze web server logs efficiently. By implementing partitioning, we optimize query performance, making log analysis scalable for large datasets.

