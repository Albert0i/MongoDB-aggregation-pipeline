### MongoDB aggregation pipeline <br />─── "Don't worry about being old, worry about thinking old"

![alt Mr.Grewgious has his suspicions](img/Mr.GrewgiousHasHisSuspicions.jpg)


### Prologue 
> —both when the tide ebbed, and when it flowed again—


### I. Quiz 
1. How many users are active ?
2. What is the average age of all users ?
3. List the top 5 most common favorite fruits among the users ?
4. Find the total number of male and female ?
5. Which country has the highest number of registered user ?
6. List all unique eye color present in the collection ?
7. What is the average number of tags per user ? 
8. How many users have 'enim' as one of their tags ? 
9. What are the name and age of user who are inactive and have 'velit' as a tag ? 
10. How many users have a phone number starting with '+1 (940)' ? 
11. Who has registered the most recently ? 
12. Categorize users by their favorite fruit ? 
13. How many users have 'ad' as the second tag in their list of tags ? 
14. Find user have both 'enim' and 'id' as their tag ? 
15. List all the companies located in USA with their corresponding user count ? 
16. Lookup exercise. 
17. List all active users with tags named sorted. 

### II. Semantics

[Relational Database](https://www.oracle.com/database/what-is-a-relational-database/) brings in their unparalleled power in the complete arbitrary of connecting tables among databases without the least taint of awkwardness. All and all empowers developers in solving complicated analytic and statistic issues at large. People writes [SQL](https://www.w3schools.com/sql/sql_intro.asp) and thus thinks in SQL regularly without second thought and been tempted to profess that it is the most natural and even the only way in solving problemss. Not until the dawn of [NoSQL Database](https://aws.amazon.com/nosql/), do people aware and become palpable of the existence of other approach to organize and manipulate the data. 

First things first, to test drive the following statement against [Oracle: 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/index.html): 
```sql
EXPLAIN PLAN FOR 
SELECT sex, round(birdat/10000), count(*), round(avg(mthsal), 2) 
FROM member 
WHERE birdat > 19000101 AND sex IN ('M', 'F') 
GROUP BY sex, round(birdat/10000)
ORDER BY 1 DESC, 2; 

SELECT * FROM table(dbms_xplan.display);
```

Output: 
```
Plan hash value: 3379579560
 
-----------------------------------------------------------------------------
| Id  | Operation          | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |        | 21992 |   257K|   685   (1)| 00:00:01 |
|   1 |  SORT GROUP BY     |        | 21992 |   257K|   685   (1)| 00:00:01 |
|*  2 |   TABLE ACCESS FULL| MEMBER | 25425 |   297K|   683   (1)| 00:00:01 |
-----------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   2 - filter("BIRDAT">19000101 AND ("SEX"='F' OR "SEX"='M'))
```

Let's check an even more complicated example: 
```sql
EXPLAIN PLAN FOR
SELECT round(f2.npotwdt/100) as yearmonth, f1.ogtabv as dept,               
       sum(NPODAYHR) as dayhour, sum(NPODAYAM) as dayamount,                 
       sum(NPONIGHR) as nighthour, sum(NPONIGAM) as nightamount
FROM master f1, detail f2        
WHERE f1.npotnum=f2.npotnum AND f1.npotseq1=f2.npotseq1 AND  
      f1.npotsts IN ('V', 'T') and f2.npotsts='A' AND f1.empnum=f2.empnum AND  
      round(f2.npotwdt/10000)=2019                      
GROUP BY round(f2.npotwdt/100), f1.ogtabv              
ORDER by 1, 2;
```

Output:  
```
Plan hash value: 3591893116
 
-------------------------------------------------------------------------------
| Id  | Operation           | Name    | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------
|   0 | SELECT STATEMENT    |         |  5668 |   974K|   892   (1)| 00:00:01 |
|   1 |  SORT GROUP BY      |         |  5668 |   974K|   892   (1)| 00:00:01 |
|*  2 |   HASH JOIN         |         |  5668 |   974K|   891   (1)| 00:00:01 |
|*  3 |    TABLE ACCESS FULL| DETAIL  |  5668 |   642K|   618   (1)| 00:00:01 |
|*  4 |    TABLE ACCESS FULL| MASTER  | 39746 |  2328K|   273   (1)| 00:00:01 |
-------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   2 - access("F1"."NPOTNUM"="F2"."NPOTNUM" AND 
              "F1"."NPOTSEQ1"="F2"."NPOTSEQ1" AND "F1"."EMPNUM"="F2"."EMPNUM")
   3 - filter("F2"."NPOTSTS"='A' AND ROUND("F2"."NPOTWDT"/10000)=2023)
   4 - filter("F1"."NPOTSTS"='T' OR "F1"."NPOTSTS"='V')
 
Note
-----
   - dynamic statistics used: dynamic sampling (level=2)
```

As you can see, every thing has a COST, any SQL statement to be executed has to be *parse* (either soft-parse or hard-parse), so that an *execution plan* is devised and get executed behind the scenes. Typically, SQL statement is depicted in a form of **Denotational Semantics**, ie. you *vaguely* tell what you want without saying how; while the NoSQL counterpart is specified in **Operational Semantics**, ie. a stage-by-stage of execution plan. 

--- 
Aaddendum: 
- **Operational Semantics** (操作語義) – the execution of the language is described directly. 
- **Denotational Semantics** (指稱語義) – each phrase in the language is interpreted as a conceptual meaning that can be thought of abstractly. 
- **Axiomatic Semantics** (公理語義) – meaning to phrases is given by describing the logical axioms that apply to them.


### III. [Aggregation Pipeline](https://www.mongodb.com/docs/manual/core/aggregation-pipeline/)

An aggregation pipeline consists of one or more [stages](https://www.mongodb.com/docs/manual/reference/operator/aggregation-pipeline/#std-label-aggregation-pipeline-operator-reference) that process documents:

- Each stage performs an operation on the input documents. For example, a stage can filter documents, group documents, and calculate values.

- The documents that are output from a stage are passed to the next stage.

- An aggregation pipeline can return results for groups of documents. For example, return the total, average, maximum, and minimum values.

Similar query in MongoDB could be: 
```
const database = 'mydb';

// The current database to use.
use(database);

db.member.aggregate([  
  { // Stage 1
    $match: {
      birdat: { $gt: 19000101 },
      sex:  { $in:  ['M', 'F'] }
    }
  },
  { // Stage 2
    $addFields: {
      yearOfBirth: 
      {
        $floor: { $divide: [ "$birdat" , 10000] }
      }      
    }
  },
  { // Stage 3 
    $group: {
      _id: {sex: "$sex" , yearOfBirth: "$yearOfBirth"}, 
      count: { $sum: 1 }, 
      average: { $avg: "$mthsal" }
    }
  },
  {
    // Stage 4
    $sort: {
      "_id.sex": -1,
      "_id.yearOfBirth": 1
    }
  }
]);
```
Output: 
```
[
  {
    "_id": {
      "sex": "M",
      "yearOfBirth": 1921
    },
    "count": 1,
    "average": 1500
  },
  {
    "_id": {
      "sex": "M",
      "yearOfBirth": 1923
    },
    "count": 1,
    "average": 4052
  },
  {
    "_id": {
      "sex": "M",
      "yearOfBirth": 1924
    },
    "count": 2,
    "average": 1587.5
  },
. . . 
]
```

The second SQL yields even more complicated aggregation pipeline: 
```
const database = 'mydb';

// The current database to use.
use(database);

db.master.aggregate([  
  { // Stage 1
    $match: {
      npotsts: { $in: [ "V", "T" ] }
    }
  },
  { // Stage 2 - multiple fields lookup (from ChatGPT)
    $lookup: {
      from: "detail",
      let: {
        npotnum: "$npotnum",
        npotseq1: "$npotseq1",
        empnum: "$empnum",
      },
      pipeline: [
        {
          $match: {
            $expr: {
              $and: [
                { $eq: ["$npotnum", "$$npotnum"] },
                { $eq: ["$npotseq1", "$$npotseq1"] },
                { $eq: ["$empnum", "$$empnum"] }
              ]
            }
          }
        }
      ],
      as: "detail"
    }
  },
  { // Stage 3
    $unwind: "$detail"
  },
  { // Stage 4 
    $addFields: {
      year: { $floor: { $divide: [ "$detail.npotwdt" , 10000] } }, 
      yearMonth: { $floor: { $divide: [ "$detail.npotwdt" , 100] } }
    }
  },
  { // Stage 5
    $match: {
      year: 2019,
      "detail.npotsts": "A"
    }
  }, 
  { // Stage 6
    $group: {
      _id: {
        yearMonth: "$yearMonth", 
        ogtabv: "$ogtabv"
      },
      dayhour: {
        $sum: "$detail.npodayhr"
      }, 
      dayamount: {
        $sum: "$detail.npodayam"
      },
      nighthour: {
        $sum: "$detail.nponighr"
      }, 
      nightamount: {
        $sum: "$detail.nponigam"
      }      
    }
  },
  { // Stage 7
    $sort: {
      "_id.yearMonth": 1,
      "_id.ogtabv": 1 
    }
  }
]);
```

![alt 2000YearsLater](img/2000YearsLater.jpg)

Output:
```
[
  {
    "_id": {
      "yearMonth": 201901,
      "ogtabv": "DC1"
    },
    "dayhour": 117,
    "dayamount": 46208,
    "nighthour": 201,
    "nightamount": 93372
  },
  {
    "_id": {
      "yearMonth": 201901,
      "ogtabv": "FNC"
    },
    "dayhour": 14,
    "dayamount": 7280,
    "nighthour": 24,
    "nightamount": 14047
  },
  {
    "_id": {
      "yearMonth": 201901,
      "ogtabv": "RDX"
    },
    "dayhour": 4,
    "dayamount": 1406,
    "nighthour": 0,
    "nightamount": 0
  },
. . .   
]  
```


### IV. MongoDB aggregation operators summary

Here's a summary of some commonly used aggregation operators in MongoDB:

1. `$match`: Filters documents based on specified criteria, similar to the `find` method. It uses the MongoDB query syntax to specify the conditions.

2. `$group`: Groups documents together based on a specified key and performs aggregations on the grouped data, such as calculating totals, averages, or counting occurrences.

3. `$sort`: Sorts the input documents based on specified fields in ascending or descending order.

4. `$project`: Reshapes documents by including or excluding fields, renaming fields, or creating new computed fields. It is often used to shape the output of the aggregation pipeline.

5. `$limit`: Limits the number of documents passed to the next stage of the pipeline.

6. `$skip`: Skips a specified number of documents and passes the remaining documents to the next stage of the pipeline.

7. `$unwind`: Deconstructs an array field, creating a new document for each element of the array. This is useful when you need to perform further operations on the individual elements.

8. `$lookup`: Performs a left outer join between two collections and enriches the input documents with matching documents from the "joined" collection.

9. `$sum`: Calculates the sum of numeric values within a group.

10. `$avg`: Calculates the average of numeric values within a group.

11. `$max` and `$min`: Retrieve the maximum and minimum values within a group, respectively.

12. `$addToSet`: Creates an array of unique values from a specified field. Duplicates are eliminated.

13. `$push`: Appends a specified value to an array field.

These are just a few examples of the aggregation operators available in MongoDB. They provide powerful capabilities for data transformation, grouping, and analysis, allowing for complex data manipulations within the aggregation framework. (generated by ChatGPT)


### V. MongoDB aggregation accumulator summary 

```
Accumulator      Description                                             Example Usage
----------------------------------------------------------------------------------------
$sum             Calculates the sum of numeric values within a group.    { $sum: <expression> }
$avg             Calculates the average of numeric values within a group. { $avg: <expression> }
$min             Finds the minimum value within a group.                  { $min: <expression> }
$max             Finds the maximum value within a group.                  { $max: <expression> }
$first           Returns the first value within a group.                  { $first: <expression> }
$last            Returns the last value within a group.                   { $last: <expression> }
$addToSet        Adds unique values to an array within a group.           { $addToSet: <expression> }
$push            Appends values to an array within a group.               { $push: <expression> }
$stdDevPop       Calculates the population standard deviation.            { $stdDevPop: <expression> }
$stdDevSamp      Calculates the sample standard deviation.                { $stdDevSamp: <expression> }
$size            Returns the size of an array within a group.              { $size: <expression> }
$sumArray        Calculates the sum of an array of numeric values.        { $sum: <arrayExpression> }
$avgArray        Calculates the average of an array of numeric values.    { $avg: <arrayExpression> }
$minArray        Finds the minimum value in an array.                     { $min: <arrayExpression> }
$maxArray        Finds the maximum value in an array.                     { $max: <arrayExpression> }
```
(generated by ChatGPT)


### VI. Introspection 
SQL database is well known for it's performance and flexibility in handling large amount of data, but there is a huge cost underneath. Database tables have to be maintained in [Normal form](https://en.wikipedia.org/wiki/Database_normalization); Schema must be simple and rigid and thus suffers from capricious change. However, in NoSQL Database, *data to be used together should be stored together* so that no additional table joining or lookup is required. In addition, data field can be object or array of objects to avoid here and there scattered lookup everywhere...

Seasoned full-stacked developers would twist SQL statements as much as possible so as to alleviate drudgery on front-end development. For sure, aggregation pipeline is the new battlefield for back-end developers and continues to play an important part in NoSQL Database. 


### VII. Reference
1. [Complete MongoDB aggregation pipeline course | Hitesh Choudhary](https://youtu.be/vx1C8EyTa7Y)

2. [Working with MongoDB in Visual Studio Code](https://code.visualstudio.com/docs/azure/mongodb)

3. [MongoDB $match (aggregation)](https://www.mongodb.com/docs/manual/reference/operator/aggregation/match/)

4. [MongoDB $group (aggregation)](https://www.mongodb.com/docs/manual/reference/operator/aggregation/group/)

5. [MongoDB $sort (aggregation)](https://www.mongodb.com/docs/manual/reference/operator/aggregation/sort/)

6. [MongoDB $lookup (aggregation)](https://www.mongodb.com/docs/manual/reference/operator/aggregation/lookup/)

7. [How do I display and read the execution plans for a SQL statement](https://blogs.oracle.com/optimizer/post/how-do-i-display-and-read-the-execution-plans-for-a-sql-statement)

8. [The Mystery of Edwin Drood](https://www.gutenberg.org/cache/epub/564/pg564-images.html)


### Epilogue

While preparing aggregation pipeline examples, three tables have been converted from Oracle to MongoDB. Imitating selection, grouping and sorting criterias which is an *unfair* comparison for no advanced features on MongoDB is used. Relational database advocates to separate tables as small as possible while NoSQL Database encourages embedding objects in schema design. This is because traditional programming languages can't handle database field of object or array of object properly. Once upon a time, there were [Object-Oriented Database](https://phoenixnap.com/kb/object-oriented-database) and [Object-Relational Database](https://www.tutorialspoint.com/object-and-object-relational-databases), but they are not on the same level of NoSQL Database. 

### PS: Original version of example1 

```
EXPLAIN PLAN FOR 
SELECT distinct round(f1.birdat/10000) AS YearOfBirth,        
       (SELECT round(avg(mthsal)) FROM member f2              
        WHERE round(f1.birdat/10000)=round(f2.birdat/10000) AND 
        f2.sex='M') AS M_AVG_MTHSAL,                        
       (SELECT round(avg(mthsal)) from memsoc f3              
        WHERE round(f1.birdat/10000)=round(f3.birdat/10000) AND 
        f3.sex='F') AS F_AVG_MTHSAL                         
FROM member f1                                              
WHERE f1.birdat > 19000101                                  
ORDER by 1; 

SELECT * FROM table(dbms_xplan.display);

Plan hash value: 691171458
 
--------------------------------------------------------------------------------------------
| Id  | Operation               | Name     | Rows  | Bytes |TempSpc| Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT        |          | 18932 |  1072K|       |  9689K  (1)| 00:06:19 |
|   1 |  SORT UNIQUE            |          | 18932 |  1072K|    32G|  7160K  (1)| 00:04:40 |
|*  2 |   HASH JOIN RIGHT OUTER |          |   510M|    27G|       |  3726  (46)| 00:00:01 |
|   3 |    VIEW                 | VW_SSQ_1 | 13652 |   346K|       |   685   (1)| 00:00:01 |
|   4 |     HASH GROUP BY       |          | 13652 |   159K|       |   685   (1)| 00:00:01 |
|*  5 |      TABLE ACCESS FULL  | MEMBER   | 16757 |   196K|       |   683   (1)| 00:00:01 |
|*  6 |    HASH JOIN RIGHT OUTER|          |  3739K|   114M|       |  1380   (2)| 00:00:01 |
|   7 |     VIEW                | VW_SSQ_2 | 12082 |   306K|       |   685   (1)| 00:00:01 |
|   8 |      HASH GROUP BY      |          | 12082 |   141K|       |   685   (1)| 00:00:01 |
|*  9 |       TABLE ACCESS FULL | MEMBER   | 14318 |   167K|       |   683   (1)| 00:00:01 |
|* 10 |     TABLE ACCESS FULL   | MEMBER   | 30953 |   181K|       |   683   (1)| 00:00:01 |
--------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   2 - access("ITEM_1"(+)=ROUND("F1"."BIRDAT"/10000))
   5 - filter("F3"."SEX"='F')
   6 - access("ITEM_2"(+)=ROUND("F1"."BIRDAT"/10000))
   9 - filter("F2"."SEX"='M')
  10 - filter("F1"."BIRDAT">19000101)
```

### EOF (2024/02/23)
