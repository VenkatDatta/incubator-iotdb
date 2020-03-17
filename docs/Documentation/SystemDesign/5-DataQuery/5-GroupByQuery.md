<!--

    Licensed to the Apache Software Foundation (ASF) under one
    or more contributor license agreements. See the NOTICE file
    distributed with this work for additional information
    regarding copyright ownership. The ASF licenses this file
    to you under the Apache License, Version 2.0 (the
    "License"); you may not use this file except in compliance
    with the License. You may obtain a copy of the License at

        http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing,
    software distributed under the License is distributed on an
    "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
    KIND, either express or implied. See the License for the
    specific language governing permissions and limitations
    under the License.

-->

# Downsampling query

* org.apache.iotdb.db.query.dataset.groupby.GroupByEngineDataSet

The result set of the downsampling query will inherit `GroupByEngineDataSet`, this class contains the following fields:
* protected long queryId
* private long interval
* private long slidingStep

The following two fields are for the entire query.
* private long startTime
* private long endTime

The following fields are for the current segment, and the time period is left-closed-right-open, that is, [[curStartTime, curEndTime) `
* protected long curStartTime;
* protected long curEndTime;
* private int usedIndex;
* protected boolean hasCachedTimeInterval;


The core method of `GroupByEngineDataSet` is very easy. First, determine if there is a next segment based on whether there is a cached time period, and return` true`; if not, calculate the segment start time and increase `usedIndex` by 1. If the segment start time has exceeded the query end time, return `false`; otherwise, calculate the query end time, set` hasCachedTimeInterval` to `true`, and return` true`:
```
protected boolean hasNextWithoutConstraint () {
  if (hasCachedTimeInterval) {
    return true;
  }

  curStartTime = usedIndex * slidingStep + startTime;
  usedIndex ++;
  if (curStartTime <endTime) {
    hasCachedTimeInterval = true;
    curEndTime = Math.min (curStartTime + interval, endTime);
    return true;
  } else {
    return false;
  }
}
```

## Downsampling query without value filter

The downsampling query logic without value filter is mainly in the `GroupByWithoutValueFilterDataSet` class, which inherits` GroupByEngineDataSet`.


This class has the following key fields:
* private Map <Path, GroupByExecutor> pathExecutors classifies aggregation functions for the same `Path` and encapsulates them as` GroupByExecutor`,
`GroupByExecutor` encapsulates the data calculation logic and method of each` Path`, which will be described later

* private TimeRange timeRange encapsulates the time interval of each calculation into an object, which is used to determine whether `Statistics` can directly participate in the calculation
* private Filter timeFilter Generates a user-defined query interval as a `Filter` object, which is used to filter the available` files`, `chunk`, and` page`
  
First, in the initialization `initGroupBy ()` method, the `timeFilter` is calculated based on the expression, and` GroupByExecutor` is generated for each `path`.

The `nextWithoutConstraint ()` method calculates the aggregate value `aggregateResults` of all aggregation methods in each` Path` by calling the `GroupByExecutor.calcResult ()` method.
The following method is used to convert the result list into a RowRecord. Note that when there are no results in the list, add `null` to the RowRecord:
```
for (AggregateResult res: fields) {
  if (res == null) {
    record.addField (null);
    continue;
  }
  record.addField (res.getResult (), res.getResultDataType ());
}
```


### GroupByExecutor
Encapsulating the calculation method of all aggregate functions under the same path, this class has the following key fields:
* Private IAggregateReader reader `SeriesAggregateReader` used to read the current` Path` data
* private BatchData preCachedData Each time the data read from `Reader` is a batch, it is likely to exceed the current time period, then this` BatchData` will be cached for the next use
* private List <Pair <AggregateResult, Integer >> results stores all the aggregation methods in the current `Path`,
For example: `select count (a), sum (a), avg (b)`, `count` and` sum` methods are stored together.
The `Integer` on the right is used to convert the result set to the order of the user query before converting it to RowRecord.

#### Main methods

```
// Read data from the reader and calculate the main method of this class.
private List <Pair <AggregateResult, Integer >> calcResult () throws IOException, QueryProcessException;

// Add the aggregation operation of the current path
private void addAggregateResult (AggregateResult aggrResult, int index);

// Determine whether the current path has completed all aggregation calculations
private boolean isEndCalc ();

// Calculate the results from the BatchData that was not used up in the last calculation
private boolean calcFromCacheData () throws IOException;

// Calculate using BatchData
private void calcFromBatch (BatchData batchData) throws IOException;

// Calculate results directly using Page or Chunk's Statistics
private void calcFromStatistics (Statistics statistics) throws QueryProcessException;

// Clear all calculation results
private void resetAggregateResults ();

// traverse and calculate the data in the page
private boolean readAndCalcFromPage () throws IOException, QueryProcessException;

```

In GroupByExecutor, because different aggregate functions of the same path use the same data, the entry method calcResult is responsible for reading all the data of the Path.
The retrieved data then calls the `calcFromBatch` method to complete the calculation of` BatchData` through all the aggregate functions.

The `calcResult` method returns all AggregateResult under the current Path and the position of the current aggregated value in the user query order. Its main logic is:

```
// Calculate the data left over from the last time, and end the calculation if you can get the result directly
if (calcFromCacheData ()) {
    return results;
}

// Because a chunk contains multiple pages, the page of the current chunk must be used up before the next chunk is opened
if (readAndCalcFromPage ()) {
    return results;
}

// If the remaining data is calculated, open a new chunk to continue the calculation
while (reader.hasNextChunk ()) {
    Statistics chunkStatistics = reader.currentChunkStatistics ();
      // determine whether statistics can be used and perform calculations
       ....
      // skip the current chunk
      reader.skipCurrentChunk ();
      // End the calculation if all results have been obtained
      if (isEndCalc ()) {
        return true;
      }
      continue;
    }
    // If you cannot use chunkStatistics, you need to use page data to calculate
    if (readAndCalcFromPage ()) {
      return results;
    }
}
```

The `readAndCalcFromPage` method is to obtain the page data from the currently opened chunk and calculate the aggregate result. Returns true when all calculations are completed, otherwise returns false. The main logic:

```
while (reader.hasNextPage ()) {
    Statistics pageStatistics = reader.currentPageStatistics ();
    // PageStatistics can only be used when the page does not intersect with other pages
    if (pageStatistics! = null) {
        // determine whether statistics can be used and perform calculations
        ....
        // skip the current page
        reader.skipCurrentPage ();
        // End the calculation if all results have been obtained
        if (isEndCalc ()) {
          return true;
        }
        continue;
      }
    }
    // When Statistics cannot be used, all data can only be taken out for calculation
    BatchData batchData = reader.nextPage ();
    if (batchData == null ||! batchData.hasCurrent ()) {
      continue;
    }
    // If the page just opened exceeds the time range, cache the data fetched and end the calculation directly
    if (batchData.currentTime ()> = curEndTime) {
      preCachedData = batchData;
      return true;
    } <!--

    Licensed to the Apache Software Foundation (ASF) under one
    or more contributor license agreements. See the NOTICE file
    distributed with this work for additional information
    regarding copyright ownership. The ASF licenses this file
    to you under the Apache License, Version 2.0 (the
    "License"); you may not use this file except in compliance
    with the License. You may obtain a copy of the License at

        http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing,
    software distributed under the License is distributed on an
    "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
    KIND, either express or implied. See the License for the
    specific language governing permissions and limitations
    under the License.

-->

# Aggregation query

The main logic of the aggregation query is in AggregateExecutor

* org.apache.iotdb.db.query.executor.AggregationExecutor

## Aggregation query without value filter

For aggregate queries without value filters, the results are obtained by the `executeWithoutValueFilter ()` method and a dataSet is constructed. First use the `mergeSameSeries ()` method to merge aggregate queries for the same time series, for example: if you need to calculate count (s1), sum (s2), count (s3), sum (s1), you need to calculate two of s1 Aggregating the values, then the pathToAggrIndexesMap result will be: s1-> 0, 3; s2-> 1; s3-> 2.

Then you will get `pathToAggrIndexesMap`, where each entry is an aggregate query of series, so you can calculate its aggregate value` aggregateResults` by calling the `groupAggregationsBySeries ()` method. Before you finally create the result set, you need to restore its order to the order of the user query. Finally use the `constructDataSet ()` method to create a result set and return it.

The `groupAggregationsBySeries ()` method is explained in detail below. First create an `IAggregateReader`:
```
IAggregateReader seriesReader = new SeriesAggregateReader (
        pathToAggrIndexes.getKey (), tsDataType, context, QueryResourceManager.getInstance ()
        .getQueryDataSource (seriesPath, context, timeFilter), timeFilter, null);
```

For each entry (that is, series), first create an aggregate result `AggregateResult` for each aggregate query, and maintain a boolean list` isCalculatedList`, corresponding to whether each `AggregateResult` has been calculated and record the remaining The number of calculated aggregate functions `remainingToCalculate`. The list of boolean values ​​and this count value will make some aggregate functions (such as `FIRST_VALUE`) do not need to continue the entire loop process after obtaining the result.

Next, update AggregateResult according to the usage method of aggregateReader introduced in Section 5.2:

```
while (aggregateReader.hasNextChunk ()) {
  if (aggregateReader.canUseCurrentChunkStatistics ()) {
    Statistics chunkStatistics = aggregateReader.currentChunkStatistics ();
    
    // do some aggregate calculation using chunk statistics
    ...
    
    aggregateReader.skipCurrentChunk ();
    continue;
  }
Ranch
  while (aggregateReader.hasNextPage ()) {
if (aggregateReader.canUseCurrentPageStatistics ()) {
Statistics pageStatistic = aggregateReader.currentPageStatistics ();
Ranch
// do some aggregate calculation using page statistics
      ...
Ranch
aggregateReader.skipCurrentPage ();
continue;
} else {
BatchData batchData = aggregateReader.nextPage ();
// do some aggregate calculation using batch data
      ...
}
  }
}
```

It should be noted that before updating each result, you need to first determine whether it has been calculated (using the isCalculatedList list); after each update, call the isCalculatedAggregationResult () method to update the boolean values ​​in the list at the same time . If all values ​​in the list are true, that is, the value of `remainingToCalculate` is 0, it proves that all aggregate function results have been calculated and can be returned.
```
if (Boolean.FALSE.equals (isCalculatedList.get (i))) {
  AggregateResult aggregateResult = aggregateResultList.get (i);
  ... // update
  if (aggregateResult.isCalculatedAggregationResult ()) {
    isCalculatedList.set (i, true);
    remainingToCalculate--;
    if (remainingToCalculate == 0) {
      return aggregateResultList;
    }
  }
}
```
When using `overlapedPageData` to update, since each batch function result will traverse this batchData, you need to call the` resetBatchData () `method to point the pointer to its starting position, so that the next function can traverse.

## Aggregation query with value filter
For aggregate queries with value filters, obtain the results through the `executeWithoutValueFilter ()` method and build a dataSet. First create a `timestampGenerator` according to the expression, then create a` SeriesReaderByTimestamp` for each time series and place it in the `readersOfSelectedSeries` list; create an aggregate result` AggregateResult` for each query and place it in the `aggregateResults` list .

After initialization is complete, call the `aggregateWithValueFilter ()` method to update the result:
```
while (timestampGenerator.hasNext ()) {
  // generate timestamps
  long [] timeArray = new long [aggregateFetchSize];
  int timeArrayLength = 0;
  for (int cnt = 0; cnt <aggregateFetchSize; cnt ++) {
    if (! timestampGenerator.hasNext ()) {
      break;
    }
    timeArray [timeArrayLength ++] = timestampGenerator.next ();
  }

  // Calculate aggregate results using timestamps
  for (int i = 0; i <readersOfSelectedSeries.size (); i ++) {
    aggregateResults.get (i) .updateResultUsingTimestamps (timeArray, timeArrayLength,
      readersOfSelectedSeries.get (i));
    }
  }
```

    // perform calculations
    calcFromBatch (batchData);
    ...
}

```

The `calcFromBatch` method is to traverse all the aggregate functions to calculate the retrieved BatchData. The main logic is:

```
for (Pair <AggregateResult, Integer> result: results) {
    // If a function has completed the calculation, it will not be calculated, such as the minimum calculation
    if (result.left.isCalculatedAggregationResult ()) {
      continue;
    }
    // perform calculations
    ....
}
// Judge whether the data in the current batchdata can still be used next time, if it can be added to the cache
if (batchData.getMaxTimestamp ()> = curEndTime) {
    preCachedData = batchData;
}
```

## Aggregation query with value filter
The downsampling query logic with value filtering conditions is mainly in the `GroupByWithValueFilterDataSet` class, which inherits` GroupByEngineDataSet`.

This class has the following key fields:
* private List <IReaderByTimestamp> allDataReaderList
* private GroupByPlan groupByPlan
* private TimeGenerator timestampGenerator
* private long timestamp is used to cache timestamp for the next group by partition
* private boolean hasCachedTimestamp is used to determine whether there is a timestamp cache for the next group by partition
* private int timeStampFetchSize is the size of the group by calculation batch

First, in the initGroupBy () method, create a timestampGenerator according to the expression; then create a SeriesReaderByTimestamp for each time series and place it in the allDataReaderList list

After initialization is complete, call the nextWithoutConstraint () method to update the result. If the timestamp is cached for the next group by partition and the time meets the requirements, add it to `timestampArray`, otherwise return the result of` aggregateResultList` directly; if the timestamp is not cached for the next group by partition, use `timestampGenerator `Traversing:

```
while (timestampGenerator.hasNext ()) {
  // call the constructTimeArrayForOneCal () method to get the timestamp list
  timeArrayLength = constructTimeArrayForOneCal (timestampArray, timeArrayLength);

  // Invoke the updateResultUsingTimestamps () method to calculate the aggregate result using the timestamp list
  for (int i = 0; i <paths.size (); i ++) {
    aggregateResultList.get (i) .updateResultUsingTimestamps (
        timestampArray, timeArrayLength, allDataReaderList.get (i));
  }

  timeArrayLength = 0;
  // determine if it is over
  if (timestamp> = curEndTime) {
    hasCachedTimestamp = true;
    break;
  }
}
```

The `constructTimeArrayForOneCal ()` method traverses timestampGenerator to build a list of timestamps:

```
for (int cnt = 1; cnt <timeStampFetchSize && timestampGenerator.hasNext (); cnt ++) {
  timestamp = timestampGenerator.next ();
  if (timestamp <curEndTime) {
    timestampArray [timeArrayLength ++] = timestamp;
  } else {
    hasCachedTimestamp = true;
    break;
  }
}
```