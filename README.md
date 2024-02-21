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


### V. Introspection 
SQL database is well known for it's performance and flexibility in handling large amount of data, but there is a huge cost underneath. Database tables have to be maintained in [Normal form](https://en.wikipedia.org/wiki/Database_normalization); Schema must be simple and rigid and thus suffers from capricious change. However, in NoSQL database, *data to be used together should be stored together* so that no additional table joining or lookup is required. In addition, data field can be object or array of objects to avoid here and there scattered lookup everywhere...

Seasoned full-stacked developers would twist SQL statements as much as possible so as to alleviate drudgery on front-end development. For sure, aggregation pipeline is the new battlefield for back-end developers and continues to play an important part in NoSQL database. 


### VI. Reference
1. [Complete MongoDB aggregation pipeline course | Hitesh Choudhary](https://youtu.be/vx1C8EyTa7Y)

2. [Working with MongoDB in Visual Studio Code](https://code.visualstudio.com/docs/azure/mongodb)

3. [MongoDB $match (aggregation)](https://www.mongodb.com/docs/manual/reference/operator/aggregation/match/)

4. [MongoDB $group (aggregation)](https://www.mongodb.com/docs/manual/reference/operator/aggregation/group/)

5. [MongoDB $sort (aggregation)](https://www.mongodb.com/docs/manual/reference/operator/aggregation/sort/)

6. [MongoDB $lookup (aggregation)](https://www.mongodb.com/docs/manual/reference/operator/aggregation/lookup/)

7. [How do I display and read the execution plans for a SQL statement](https://blogs.oracle.com/optimizer/post/how-do-i-display-and-read-the-execution-plans-for-a-sql-statement)

8. [The Mystery of Edwin Drood](https://www.gutenberg.org/cache/epub/564/pg564-images.html)


### Epilogue


### EOF (2024/02/21)
