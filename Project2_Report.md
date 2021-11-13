# Project 2 Report

## Pipeline Overview
* I used docker-compose to manage a cluster of containers- Zookeeper, Kafka, Cloudera, Spark, and the MIDS container. Each container is responsible for a different stage in the pipeline.
* Zookeeper acts as the "phonebook" for Kafka to help manage the cluster. Kafka ingests the data by producing and consuming each data record as messages. These Kafka messages are then transformed into queryable dataframes using Spark (PySpark and SparkSQL). Cloudera is the host for HDFS, where the transformed data is landed for others to access and query. The MIDS container provides tools such as kafkacat and helps process json files using pre-downloaded tools.
* I landed two dataframes into HDFS. One dataframe contains records of each distinct assessment attempt, and it allows queries for identifying the most/least common courses, average performance of a course, and the number of assessments/attempts (I will use "assessment" and "attempt" synonymously throughout). The other dataframe contains records of each distinct question on an exam (note: same questions on different exams are considered different) and allows queries about each question such as identifying the question that most people got wrong, most commonly seen question.

## Git Repo Structure & docker-compose.yml Details
The Git repo is a separate branch from master containing the markdown file for report, a text file showing command line history, and a docker-compose.yml file. The docker-compose.yml file is copied from Week 8's content.
Inside the docker-compose.yml file contains all the details for the 5 containers used to spin up the pipeline:
* zookeeper: the "phonebook" for Kafka
  * zookeeper's "image" tells us the name of zookeeper's image and it's currently using the latest version.
  * the ports listed under "expose" allows other containers to communicate with zookeeper. For example, kafka uses the port "32181" to connect to zookeeper.
  * kafka connects to zookeeper via "32181" because the ZOOKEEPER_CLIENT_PORT specified under environment tells us that 32181 is the connection.
* kafka: the "message broker"
  * kafka's "image" tells us the name of the kafka image and that we're using the latest version.
  * "depends_on" tells us that kafka depends on zookeeper and thus requires zookeeper to be up before it can start running.
  * kafka knows to connect to zookeeper using "zookeeper:32181" because it's specified under KAFKA_ZOOKEEPER_CONNECT
  * kafka is also listening and other containers can connect to kafka via "kafka:29092" outlined under KAFKA_ADVERTISED_LISTENERS
* cloudera: host for HDFS
* spark: for data processing & filtering
  * "depends_on" cloudera- needs cloudera to be spun up before it can start running
  * the "image" tells us that it's using version 0.0.5 of spark-python
  * stdin_open and tty set to true allows an interactive mode, so we can use a terminal to access Spark
  * "volumes" allows us to connect the host system directory with the container. The volumes option says "~/w205:/w205", which tells us we can access anything in the w205 folder by simply referring to the path as "/w205".
  * "HADOOP_NAMENODE: cloudrea" establishes the connection between Spark and HDFS, so Spark knows that we're writing to HDFS without specifying that connection.
* mids: provides extra tools (eg. kafkacat)!
  * can run a terminal on this container because stdin_open and tty are set to true

## Overview of Data File Strucutre
The json file contains 3280 records of assessment/attempts from various users on various courses. Description of the file's fields are my own assumptions. The data follows this structure:
* keen_timestamp: (type string) timestamp for "keen". Not sure what "keen" is.
* max_attempts: (type string) the max number of attempts allowed on this exam
* started_at: (type string) the time at which the assessment is started ("assessment" and "attempt" will be used interchangeably)
* base_exam_id: (type string) exam's id; unique to each exam/course ("exam" and "course will be used interchangeably)
* user_exam_id: (type string) unique id for each user
* sequences: (type map) contains all the questions for this assessment. Extremely nested.
  * questions: (nested) each question in the exam
    * 0: question #
      * user_incomplete: (type boolean) whether or not the user completed this question on this attempt
      * user_correct: (type boolean) whether or not the user got this question right on this attempt
      * options: (nested) all the checkable options on this question; contains details about whether each option is checked, answered correctly, etc.
      * user_submitted: (type boolean) whether or not the user submitted this question on this attempt
      * id: (type string) unique id for this question
      * user_result: (type string) a short description of the result for this question on this assessment (eg. "correct")
    * ...
    * n:
  * attempt: (type long) attempt #
  * id: (type string) ID for this series of questions
  * counts: (nested) contains numeric data related to the performance of this attempt
    * incomplete: (type long) number of questions that are incomplete
    * submitted: (type long) number of questions submitted
    * incorrect: (type long) number of questions the user got wrong on this attempt
    * all_correct: (type boolean) whether or not the user got all the questions right
    * correct: (type long) number of questions the user got right on this attempt
    * total: (type long) number of questions in this exam
    * unanswered: (type long) number of questions the user did not answer
* keen_created_at: (type string) timestamp at which "keen" was created. Seems like it's the same value as keen_timestamp
* certification: (type string) whether the attempt resulted in a certification, or whether the exam offers a certification. They're all false anyway (except missing values)
* keen_id: (type string) id for "keen", perhaps referring to the database this data was stored at / retrieved from
* exam_name: (type string) name of the exam/course

## Pipeline Commands in Detail

#### *Step 1: Make & switch to the Project 2 directory. Clone project 2 git repo. Create and checkout to a new branch*

### Publish and consume messages with Kafka (steps 2-6)

#### *Step 2: copy docker-compose.yml file from Week 8 synchrnous content*
```
cp ../course-content/08-Querying-Data/docker-compose.yml .
```

#### *Step 3: spin up docker-compose in background*
```
docker-compose up -d
```
command explanation:
* -d stands for "detached," and it allows docker-compose to run in the background so the terminal can be used while docker-compose is running.

#### *Step 4: Download the data (used code from project 2 README*

#### *Step 5: Create a Kafka topic*
```
docker-compose exec kafka kafka-topics --create --topic exam-attempts --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:32181
```
command explanation:
* `exec kafka`: execute kafka container
* `kafka-topics --create --topic <topic-name>`: use the kafka-topics tools in Kafka to create a topic called "exam-attempts." I named it the topic "exam-attempts" because the messages I produced within this topic are assessment attempts for various exams.
* `--partitions 1 --replication-factor 1 --if-not-exists`: setup for the Kafka topic (1 partition, replication factor of 1, and suprress warning)
* `--zookeeper zookeeper:32181`: tells Kafka to connect to zookeeper via the zookeerper connection port "zookeeper:32181".

#### *Step 6: Produce messages with Kafka*
```c
docker-compose exec mids bash -c "cat /w205/project-2-MeerWu/assessment-attempts-20180128-121051-nested.json | jq '.[]' -c | kafkacat -P -b kafka:29092 -t exam-attempts && echo 'Produced 3280 messages.'"
```
command explanation:
* `docker-compose exec mids bash -c`: in docker-compose we want to execute the mids container and use bash. -c indicates we want to type a command line after.
* `cat /w205/project-2-MeerWu/assessment-attempts-20180128-121051-nested.json | jq '.[]' -c`: this is a command line prompt. Take the contents of json file downloaded earlier (which is the output of the code before `|`) and input it to a a json tool (jq) via the `|` command to unwrap the outer layer brackets (`'.[]'`) and read in each record on a separate line via the `-c` command.
* ` | kafkacat -P -b kafka:29092 -t exam-attempts && echo 'Produced 3280 messages.'`: Take each record per line (output of previous commands) as input and use kafkacat to produce (`-P`) messages with Kafka. Specify the bootstrap server via the `-b` command and the Kafka topic to write to via the `-t` command. On top of that command, print a message that says "Produced 3280 messages" when done.

### Use Spark to transform the messages (steps 7-15)

#### *Step 7: Start interactive PySpark in terminal*
```c
docker-compose exec spark pyspark
```
command explanation: execute the spark container and use PySpark, a tool in Spark.

#### *Step 8: Read & load data [in PySpark]*
```c
attempts = spark.read.format("kafka").option("kafka.bootstrap.servers", "kafka:29092").option("subscribe", "exam-attempts").option("startingOffsets", "earliest").option("endingOffsets", "latest").load()
```
command explanation:
* `spark.read`: tell Spark we're reading something
* `.format("kafka")`: read from kafka container
* `.option("kafka.bootstrap.servers", "kafka:29092")`: connect to Kafka bootstrap server via "kafka:29092"
* `.option("subscribe", "exam-attempts")`: subscribe to the topic named "exam-attempts"
* `.option("startingOffsets", "earliest").option("endingOffsets", "latest")`: want to read all the data so the starting offset = earliest message & ending offset = latest message produced
* `.load()`: load the messages into a DataFrame

#### *Step 9: Check schema and size of datafrmae [in PySpark]*
```c
attempts.show()
attempts.printSchema()
attempts.count()
```
command explanation:
* 1st line gives a sneak-peek of what the dataframe looks like. `attempts` is adataframe with a **key** and **value** column, where both columns encode binary values, and each row contains a record.
* 2nd line shows the schema of the data.
* 3rd line shows the number of rows in the dataframe. It should match the number of messages produced (and it did).

#### *Step 10: Cast & Cache, import statements, and Encoding in utf8 [in PySpark]*
```c
attempts = attempts.selectExpr("CAST(key AS STRING)", "CAST(value AS STRING)")
attempts.printSchema()
attempts.show()
attempts.cache()
from pyspark.sql import Row
import json
import sys
sys.stdout = open(sys.stdout.fileno(), mode='w', encoding='utf8', buffering=1)
```
command explanation:
* 1st line casts the key and value as strings.
* 2nd & 3rd line check what the newly casted types are. `attempts` is now a dataframe with the **key** and **value** column are of type string.
* 4th line caches the dataframe so we run into less errors and make queries run faster.
* last line encodes the data in utf8.

#### *Step 11: Build functions for extracting nested json data*
I built two functions for unrolling the nested json data:
* the first one for data within each question nested in the **sequences** key and
* the second for getting data in within **counts**, which was nested in the **sequences** key.
 
I needed to use these functions because data inside **sequences** would be nested in a Map object that did not preserve the structure of the original json file if I just read in each record as is (the map object had **questions** as key and the value was another map object the first key within each question as its key). These functions allowed me to retrieve nested values in the correct format and handle missing values within nested data. If a key does not exist, Spark will default the return value to null, but it throws a KeyError instead if a key along the path of keys used to get nested data does not exist for some records. To avoid this error, I only retreieved the nested values if the nested key existed. Otherwise, the value is defaulted to -99. The main difference between the 1st & 2nd function is that the first returns a list of Rows while the second returns a single Row. The first function returns a list of Rows because I need to get multiple rows worth of data from each record for the dataframe with each question as a row.
```c
def extract_question_details(row):
    #load the value of each row, where each value is a record in the json file
    attempts = json.loads(row.value)
    
    #get all nested key-value pairs in 'questions' for this row if 'questions' is a valid key within 'sequences'
    questions_rows = []
    if "sequences" in attempts.keys():
        if "questions" in attempts["sequences"].keys():
            questions = attempts["sequences"]["questions"]
    
    #get the data fields needed; default the values nested within "questions" to -99 to avoid KeyError
    for q in questions:
        q_dict = {"question_id": q["id"],
                  "question_incomplete": -99,
                  "question_correct": -99,
                  "question_submitted": -99,
                  "exam_id": attempts["base_exam_id"],
                  "exam_name": attempts["exam_name"],
                  "user_id": attempts["user_exam_id"],
                  "start_time": attempts["started_at"]}
        if "user_incomplete" in q.keys():
            q_dict["question_incomplete"] = q["user_incomplete"]
        if "user_correct" in q.keys():
            q_dict["question_correct"] = q["user_correct"]
        if "user_submitted" in q.keys():
            q_dict["question_submitted"] = q["user_submitted"]
        #append data from this question to a list of questions as a sparkSQL Row
        questions_rows.append(Row(**q_dict))
    #returns a list of SparkSQL Rows
    return questions_rows
```
```c
def extract_count_details(row):
    #load the value of each row, where each value is a record in the json file
    attempts = json.loads(row.value)
    
    #get the data fields needed; nested values that do not exist default to -99
    count_details = {"exam_id": attempts["base_exam_id"],
                     "certification": attempts["certification"],
                     "exam_name": attempts["exam_name"],
                     "max_attempts": attempts["max_attempts"]}
    if "counts" in attempts["sequences"].keys():
        count_details["num_questions"] = attempts["sequences"]["counts"]["total"]
        count_details["num_correct"] = attempts["sequences"]["counts"]["correct"]
        count_details["num_incorrect"] = attempts["sequences"]["counts"]["incorrect"]
    else:
        count_details["num_questions"] = -99
        count_details["num_correct"] = -99
        count_details["num_incorrect"] = -99
    count_details["started_at"] = attempts["started_at"]
    count_details["user_exam_id"] = attempts["user_exam_id"]
    
    #returns a single SparkSQL Row of a dictionary with the data (nested or not)
    return Row(**count_details)
```

#### *Step 12: Read Data into DataFrame*
```c
#use the first function
questions_df = attempts.rdd.flatMap(extract_question_details).toDF()
#use the second function
attempts_df = attempts.rdd.map(extract_count_details).toDF()
```
command explanation:
* `attempts.rdd.map(<func>)`/`attempts.rdd.flatMap(<func>)`: get RDD (Resilient Distributed Dataset) from the data and apply a map to it
  * `map` function takes a function that returns a single SparkSQL Row object as an argument, where the function takes in a single dataframe row
  * `flatMap` function takes a function that can return multiple elements as an argument, where the function takes in a single dataframe row, so this function can flatten array elements, allowing me to pass a list of Row objects at a time.
* `.toDF()`: converts the RDD back to a Spark DataFrame

#### *Step 13: Cast the Data Fields into Workable Types*
A lot of the fields are of type string when they really are boolean or integer values. I casted boolean values in the questions dataframe to integer values so I could perform aggregate functions later in my queries. I left the **started_at** timestamp column as type string because it was easier to convert it later using SparkSQL.
```c
#type casting the questions dataframe
questions_df = questions_df.select(questions_df.exam_id, questions_df.exam_name, questions_df.question_id, questions_df.question_correct.cast('int'), questions_df.question_incomplete.cast('int'), questions_df.question_submitted.cast('int'), questions_df.start_time, questions_df.user_id)
#type casting the attempts dataframe
attempts_df = attempts_df.select(attempts_df.certification.cast("boolean"), attempts_df.exam_id, attempts_df.exam_name, attempts_df.max_attempts.cast("float"), attempts_df.num_correct.cast("int"), attempts_df.num_incorrect.cast("int"), attempts_df.num_questions.cast("int"), attempts_df.started_at, attempts_df.user_exam_id)
```
command explanation:
* `.select()`: selects columns in the questions dataframe to keep
* `.cast("int")`: cast the column to type integer. Can also pass in "string" or "boolean" to cast to type string or boolean values, respectively.

#### *Step 14: Register a Temp Table / Create a view for SparkSQL Queries*
```c
questions_df.registerTempTable("questions")
attempts_df.registerTempTable("attempts")
```
command explanation:
* `<dataframe>.registerTempTable` function registers a temporary table so we can use the name passed into its argument to perform queries on the `<dataframe>`.

#### *Step 15: Use SparkSQL to transform the dataframes into queryable format*
```
#aggregate all the questions so that each distinct question from an exam is a single row
#add up all correct attempts, incorrect attempts, incomplete attempts, submits, attempts, and distinct users for each question
question_details = spark.sql("select exam_id, exam_name, question_id, sum(question_correct) as num_correct_submits, sum(question_submitted)-sum(question_correct)-sum(question_incomplete) as num_incorrect_submits, sum(question_incomplete) as num_incomplete_submits, sum(question_submitted) as num_submits, count(question_id) as num_attempts, count(distinct user_id) as num_users from questions group by exam_id, exam_name, question_id order by exam_name")

#pick out the columns that are used
attempts_details = spark.sql("select exam_id, exam_name, num_correct, num_incorrect, num_questions, user_exam_id from attempts")
```
The dataframe with information on each question looks like this:
```
>>> question_details.show()
+--------------------+--------------------+--------------------+-------------------+---------------------+----------------------+-----------+------------+---------+
|             exam_id|           exam_name|         question_id|num_correct_submits|num_incorrect_submits|num_incomplete_submits|num_submits|num_attempts|num_users|
+--------------------+--------------------+--------------------+-------------------+---------------------+----------------------+-----------+------------+---------+
|4cdf9b5f-fdb7-4a4...|A Practical Intro...|dab47905-63c6-46b...|                  6|                    3|                     0|          9|           9|        9|
|4cdf9b5f-fdb7-4a4...|A Practical Intro...|a6effaf7-94ba-458...|                  3|                    4|                     2|          9|           9|        9|
|4cdf9b5f-fdb7-4a4...|A Practical Intro...|7ff41d87-1f73-406...|                  6|                    3|                     0|          9|           9|        9|
|4cdf9b5f-fdb7-4a4...|A Practical Intro...|80aad87e-c7b2-4e1...|                  6|                    3|                     0|          9|           9|        9|
|e824836a-3835-4e7...|Advanced Machine ...|b07d91c2-6bcf-11e...|                 53|                   14|                     0|         67|          67|       67|
|e824836a-3835-4e7...|Advanced Machine ...|47675a7a-6bce-11e...|                 29|                   25|                    13|         67|          67|       67|
|e824836a-3835-4e7...|Advanced Machine ...|daea7559-6bd0-11e...|                 52|                   13|                     0|         65|          67|       67|
|e824836a-3835-4e7...|Advanced Machine ...|2a3e4766-6bd0-11e...|                 60|                    6|                     0|         66|          67|       67|
|f0633ed7-748d-11e...|Amazon Web Servic...|83cd9b0c-d60a-441...|                  8|                    2|                     0|         10|          12|       12|
|f0633ed7-748d-11e...|Amazon Web Servic...|8c120ddf-9ae8-437...|                  7|                    1|                     2|         10|          12|       12|
```
The dataframe with information on each assessment attempt looks like this:
```
>>> attempts_details.show()
+--------------------+--------------------+-----------+-------------+-------------+--------------------+
|             exam_id|           exam_name|num_correct|num_incorrect|num_questions|        user_exam_id|
+--------------------+--------------------+-----------+-------------+-------------+--------------------+
|37f0a30a-7464-11e...|Normal Forms and ...|          2|            1|            4|6d4089e4-bde5-4a2...|
|37f0a30a-7464-11e...|Normal Forms and ...|          1|            1|            4|2fec1534-b41f-441...|
|4beeac16-bb83-4d5...|The Principles of...|          3|            1|            4|8edbc8a8-4d26-429...|
|4beeac16-bb83-4d5...|The Principles of...|          2|            0|            4|c0ee680e-8892-4e6...|
|6442707e-7488-11e...|Introduction to B...|          3|            1|            4|e4525b79-7904-405...|
|8b4488de-43a5-4ff...|        Learning Git|          5|            0|            5|3186dafa-7acf-47e...|
|e1f07fac-5566-4fd...|Git Fundamentals ...|          1|            0|            1|48d88326-36a3-4cb...|
|7e2e0b53-a7ba-458...|Introduction to P...|          5|            0|            5|bb152d6b-cada-41e...|
|1a233da8-e6e5-48a...|Intermediate Pyth...|          4|            0|            4|70073d6f-ced5-4d0...|
|7e2e0b53-a7ba-458...|Introduction to P...|          0|            0|            5|9eb6d4d6-fd1f-4f3...|
```

### Land Messages into HDFS (step 16)
To get the data ready for data scientists to query, I wrote the transformed dataframes into Parquet and land them into HDFS
#### *Step 16: Write into Parquet & land the dataframes into HDFS*
```
question_details.write.parquet("/tmp/assessment_questions")
attempts_details.write.parquet("/tmp/assessment_attempts_details")
```
To check that the files landed in HDFS, run this:
```
#check all files in tmp folder; check all folders are created
docker-compose exec cloudera hadoop fs -ls -h /tmp/

#check files within the folder created for questions dataframe then the attempts dataframe
#tmp is the default directory for HDFS
docker-compose exec cloudera hadoop fs -ls -h /tmp/assessment_questions/
docker-compose exec cloudera hadoop fs -ls -h /tmp/assessment_attempts_details/
```
command explanation: execute cloudera container and list files in HDFS in human readable format (`-h`)

## Business Insights from the Data
1. How many assesstments are in the dataset?
    * Answer: There are 3280 assessments in the dataset.
    ```
    >>> spark.sql("select count(*) as num_assessments from attempts").show()
    +---------------+
    |num_assessments|
    +---------------+
    |           3280|
    +---------------+
    ```
    This query uses the attempts dataframe and counted the number of rows in the dataframe.
    
2. What is the most commonly taken course? How many times has it been taken? What is its average performance?
    * Answer: The most common course taken is *Learning Git*, and it's been taken 394 times. The average accuracy for the assessments is 68%.
    ```
    >>> spark.sql("select exam_name, count(exam_name) as num_exams, round(avg(num_correct)/first(num_questions)*100) as avg_performance from attempts where num_questions != -99 group by exam_name order by num_exams desc limit 5").show()
    +--------------------+---------+---------------+                                
    |           exam_name|num_exams|avg_performance|
    +--------------------+---------+---------------+
    |        Learning Git|      394|           68.0|
    |Introduction to P...|      162|           57.0|
    |Introduction to J...|      158|           88.0|
    |Intermediate Pyth...|      158|           51.0|
    |Learning to Progr...|      128|           54.0|
    +--------------------+---------+---------------+
    ```
    This query uses the attempts dataframe to see which course is most commonly taken by counting the number of attempts associated with each distinct exam/course. In addition, it calculates the average performance on these courses by looking at the percentages derived from averages of the number of correctly answered questions out of each courses's total number of questions.
    
3. How many questions are in the dataset? How many of them have 100% accuracy (ie. all attempts were correct)?
    * Answer: 473 questions in the dataset. 64 of them were answered correctly every time.
    ```
    >>> question_details.registerTempTable("questions_details")
    >>> spark.sql("select count(*) as num_questions from questions_details").show()
    +-------------+                                                                 
    |num_questions|
    +-------------+
    |          473|
    +-------------+
    >>> spark.sql("select count(*) as num_100_questions from questions_details where num_correct_submits/num_submits = 1").show()
    +-----------------+                                                             
    |num_100_questions|
    +-----------------+
    |               64|
    +-----------------+
    ```
    The first query counts the number of rows in the questions dataframe. The second query counts the number of entries where all the attempts associated that question got the entire question right.
