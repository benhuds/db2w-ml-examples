# In-database machine learning with Db2 Warehouse on Cloud

**[IBM Db2 Warehouse on Cloud](https://www.ibm.com/cloud/db2-warehouse-on-cloud)
is IBM's elastic cloud data warehouse service**.  This is a simple tutorial that
walks through how to use Db2 Warehouse on Cloud's built-in machine learning
functions to train and run models on data residing in Db2 Warehouse on Cloud.
Operations are performed by the Db2 Warehouse engine itself - no data movement
required.

### What this tutorial covers

1. How to provision an IBM Db2 Warehouse on Cloud system
2. How to load some data into your Db2 Warehouse on Cloud system
3. How to train and run a simple ML model on data residing in a Db2 Warehouse on
Cloud system

### Prerequisites

For this tutorial, you'll need a Db2 Warehouse on Cloud system:
 
1. Register for an IBM Cloud account
[here](https://cloud.ibm.com/registration?target=%2Fcatalog%2Fservices%2Fdb2-warehouse).
You'll get $200 in IBM Cloud credit, which you can use towards Db2 Warehouse on
Cloud.
2. Navigate to the [Db2 Warehouse on Cloud
page](https://cloud.ibm.com/catalog/services/db2-warehouse).  Select the  _Flex
One_ plan, and click _Create_ to create your Flex One system.

Flex One is a paid Db2 Warehouse on Cloud plan, but the amount of IBM Cloud
credits incurred during this tutorial is minimal.

### Loading data

In this repository, you'll find two CSV files: `Aircraft.csv` and
`BOS-SFO-12months.csv`.  They contain manufacturing information (e.g. make,
model) on commercial aircraft and flight on-time performance respectively.  For
the purposes of this tutorial, the dataset focuses on direct flights from Boston
to San Francisco.

You can easily drag and drop these files from your desktop and into Db2
Warehouse on Cloud through the "Load" page on the Db2 Warehouse on Cloud
console.  A quick video on how to do this can be found
[here](https://www.youtube.com/watch?v=AwiHZNaGkoA).  Load each file into
a separate table.

### Simple analysis

Once you've loaded the data into Db2 Warehouse on Cloud, you can explore it
through the "Explore" page on the Db2 Warehouse on Cloud console, or use the
"Run SQL" page to query your data directly.  Here's a simple query you can run
to calculate average arrival delay for each carrier in the past 12 months:

```sql
SELECT "OP_CARRIER", AVG("ARR_DELAY") AS "AVERAGE ARRIVAL DELAY"
FROM BFD_12MONTHS
GROUP BY "OP_CARRIER"
ORDER BY "AVERAGE ARRIVAL DELAY" ASC;
```

`BFD_12MONTHS` is what I've named my table containing on-time performance
information, but you can replace that with whatever name you go with for your
own database.

### One step further: machine learning

This information can be easily visualized through any standard BI/dashboarding
tool, but let's take this a step further.  Let's use this information to try and
predict how late a flight will be in future.  When booking flights, I'd like to
make sure my flight arrives at my destination on time so I don't miss any
connections or meetings on the other end, so a tool to help me predict delays
will definitely help me make smarter flight choices.

The problem we want to solve?  Simply put, given a choice of carrier, month/date/time
of departure, and manufacturer, I'd like to know if my flight will have a
delayed arrival.  We can use a [linear
regression](https://en.wikipedia.org/wiki/Linear_regression) model to help us do
this.

First, let's enrich our on-time performance data so it contains the aircraft
manufacturer, and store it as a view.  As with our on-time data, I've named my
table containing manufacturing data `TAILNUMS` because it uses aircraft tail
numbers as the primary key.  Feel free to replace that name with whatever name
you decide to go with.

```sql
CREATE VIEW BFD_12MONTHS_VIEW
(ID, OP_CARRIER, TAIL_NUM, MANUFACTURER, MODEL,
ORIGIN, ORIGIN_CITY_NAME, DEST, DEST_CITY_NAME,
MONTH, DAY_OF_WEEK, DEP_TIME_BLK, ARR_DELAY)
AS
(SELECT
FD.ID, FD.OP_CARRIER, FD.TAIL_NUM, TD.MANUFACTURER, TD.MODEL,
FD.ORIGIN, FD.ORIGIN_CITY_NAME, FD.DEST, FD.DEST_CITY_NAME,
FD.MONTH, FD.DAY_OF_WEEK, FD.DEP_TIME_BLK, FD.ARR_DELAY
FROM
BFD_12MONTHS AS FD
INNER JOIN
TAILNUMS AS TD
ON
FD.TAIL_NUM = TD.TAIL_NUMBER);
```

After all, manufacturer may be a factor in lateness, but it's up to our ML model
to determine how much impact it really has.

Now that we've enriched our data, let's split our overall dataset into training
and test datasets.  We'll use the training data to train our ML model, then
evaluate its performance/accuracy later by running the ML model on our test
data.

```sql
CALL IDAX.SPLIT_DATA('intable=BFD_12MONTHS_VIEW,
traintable=BFD12_TRAIN,
testtable=BFD12_TEST,
id=ID,
fraction=0.35');
```

The training and test data will be stored in the `BFD12_TRAIN` and `BFD12_TEST`
tables respectively, and this function will use the `ID` column to randomly
split the data for each.

Now, let's train our linear regression model using the training data in the
`BFD12_TRAIN` table, and call the resulting model `BFD12_ARR`.  We would like to
predict arrival delay (`ARR_DELAY`), given carrier, manufacturer, model, day of
week, month, and scheduled departure time (maybe time of day affects delays
too, who knows).

```sql
CALL IDAX.LINEAR_REGRESSION
('model=BFD12_ARR,
intable=BFD12_TRAIN,
id=ID,
target=ARR_DELAY,
incolumn=OP_CARRIER; MANUFACTURER; MODEL; DAY_OF_WEEK; MONTH; DEP_TIME_BLK,
coldefrole=ignore, calculatediagnostics=false');
```

That was easy! No new frameworks to learn, no need to move the data to an
external execution engine - just one stored procedure to run.  Tell the database
what you're trying to predict, and the variables you think affect its value.

It's even easier to test your model against the test data:

```sql
CALL IDAX.PREDICT_LINEAR_REGRESSION
('model=BFD12_ARR,
intable=BFD12_TEST,
outtable=BFD12_ARR_OUT,
id=ID');
```

Essentially: "use the `BFD_12_ARR` model to predict arrival delays in
`BFD12_TEST` and store the output in `BFD12_ARR_OUT`."

Once you've run the model on your test data, let's compare the actual delays to
the delays we predicted.  Keep in mind that for this data, negative delay means
the flight landed early.

```sql
SELECT IN.ID,
IN.ARR_DELAY as SOURCE_DELAY, OUT.ARR_DELAY AS SOURCE_PREDICT
FROM BFD12_TEST AS IN, BFD12_ARR_OUT AS OUT WHERE IN.ID=OUT.ID;
```

