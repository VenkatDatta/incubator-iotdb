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

# Query by device alignment

AlignByDevicePlan The table structure corresponding to the device alignment query is:

Time | Device | sensor1 | sensor2 | sensor3 | ... |
| ---- | ------ | ------- | ------- | ------- | --- |
| | | | | | |

## Design principle

The implementation principle of the device-by-device query is mainly to calculate the measurement points and filter conditions corresponding to each device in the query, and then the query is performed separately for each device, and the result set is assembled and returned.

### Meaning of important fields in AlignByDevicePlan

First explain the meaning of some important fields in AlignByDevicePlan:
-`List <String> measurements`: The list of measurements that appear in the query.
-`Map <Path, TSDataType> dataTypeMapping`: This variable inherits from the base class QueryPlan. Its main function is to provide the data type corresponding to the path of the query when calculating the execution path of each device.
-`Map <String, Set <String >> deviceToMeasurementsMap`,` Map <String, IExpression> deviceToFilterMap`: These two fields are used to store the measurement points and filter conditions corresponding to the device.
-`Map <String, TSDataType> measurementDataTypeMap`: AlignByDevicePlan requires that the sensor data type of the same name of different devices be consistent. This field is a Map structure of` measurementName-> dataType`, which is used to verify the data type consistency of the sensor of the same name. For example `root.sg.d1.s1` and` root.sg.d2.s1` should be of the same data type.
-`enum MeasurementType`: Records three measurement types. Measurements that do not exist in any device are of type `NonExist`; measurements with single or double quotes are of type` Constant`; measurements that exist are of type `Exist`.
-`Map <String, MeasurementType> measurementTypeMap`: This field is a Map structure of` measureName-> measurementType`, which is used to record all measurement types in the query.
-groupByPlan, fillQueryPlan, aggregationPlan: To avoid redundancy, these three execution plans are set as subclasses of RawDataQueryPlan and set as variables in AlignByDevicePlan. If the query plan belongs to one of these three plans, the field is assigned and saved.

Before explaining the specific implementation process, a relatively complete example is given first, and the following explanation will be used in conjunction with this example.

```sql
SELECT s1, "1", *, s2, s5 FROM root.sg.d1, root.sg. * WHERE time = 1 AND s1 <25 ALIGN BY DEVICE
```

Among them, the time series in the system is:

-root.sg.d1.s1
-root.sg.d1.s2
-root.sg.d2.s1

The storage group `root.sg` contains two devices d1 and d2, where d1 has two sensors s1 and s2, d2 has only sensor s1, and the same sensor s1 has the same data type.

The following will be explained according to the specific process:

### Logical plan generation

-org.apache.iotdb.db.qp.Planner

Unlike the original data query, the alignment by device query does not concatenate the suffix paths in the SELECT statement and the WHERE statement at this stage, but when the physical plan is subsequently generated, the mapping value and filter conditions corresponding to each device are calculated. Therefore, the work done at this stage by device alignment only includes optimization of filter conditions in WHERE statements.

The optimization of the filtering conditions mainly includes three parts: removing the negation, transforming the disjunction paradigm, and merging the same path filtering conditions. The corresponding optimizers are: RemoveNotOptimizer, DnfFilterOptimizer, MergeSingleFilterOptimizer. This part of the logic can refer to: [Planner] (/ # / SystemDesign / progress / chap2 / sec2).

### Physical plan generation

-org.apache.iotdb.db.qp.strategy.PhysicalGenerator

After the logical plan is generated, the `transformToPhysicalPlan ()` method in the PhysicalGenerator class is called to convert the logical plan into a physical plan. For device-aligned queries, the main logic of this method is implemented in the transformQuery () method.

** The main work done at this stage is to generate ** `AlignByDevicePlan` corresponding to the query, ** fill the variable information in it. **

First explain the meaning of some important fields in the `transformQuery ()` method (see above for the duplicate fields in AlignByDevicePlan):

-prefixPaths, suffixPaths: the former is the prefix path in the FROM clause, in the example `[root.sg.d1, root.sg. *]`; the latter is the suffix path in the SELECT clause, in the example `[ s1, "1", *, s2, s5] `.
-devices: Device list obtained by removing wildcards and deduplication of the prefix path, in the example `[root.sg.d1, root.sg.d2]`.
-measurementSetOfGivenSuffix: an intermediate variable that records the measurement corresponding to a suffix. In the example, for the suffix \ *, `measurementSetOfGivenSuffix = {s1, s2}`, for the suffix s1, `measurementSetOfGivenSuffix = {s1}`;

Next, introduce the calculation process of AlignByDevicePlan:

1. Check whether the query type is one of the three types of queries: groupByPlan, fillQueryPlan, aggregationPlan. If it is, assign the corresponding variable and change the query type of `AlignByDevicePlan`.
2. Traverse the SELECT suffix path and set an intermediate variable for each suffix path as `measurementSetOfGivenSuffix` to record all measurements corresponding to the suffix path. If the suffix path starts with single or double quotes, increase the value directly in `measurements` and note that its type is` Constant`.
3. Otherwise, stitch the device list with the suffix path to get the complete path. If the stitched path does not exist, you need to further determine whether the measurement exists in other devices. If none, temporarily identify it as `NonExist`. If the measurement exists on the device, the value of `NonExist` will be overwritten by` Exist`.
4. If the path exists after splicing, prove that the measurement is of type `Exist`, you need to check the consistency of the data type, and return an error message if it is not satisfied. If it is met, record the measurement and update` measurementSetOfGivenSuffix`, `deviceToMeasurementsMap`, etc.
5. After the suffix loop of a layer ends, add the measurementSetOfGivenSuffix that appears in the loop of the layer to the measurement. At the end of the entire loop, the variable information obtained in the loop is assigned to AlignByDevicePlan. The list of measurements obtained here is not duplicated and will be de-duplicated when the ColumnHeader is generated.
6. Finally call the `concatFilterByDevice ()` method to calculate `deviceToFilterMap`, and get the corresponding Filter information after splicing each device separately.

```java
Map <String, IExpression> concatFilterByDevice (List <String> devices,
      FilterOperator operator)
Input: deduplicated devices list and unstitched FilterOperator
Input: deviceToFilterMap after splicing, which records the Filter information corresponding to each device
```

The main processing logic of the `concatFilterByDevice ()` method is in `concatFilterPath ()`:

The `concatFilterPath ()` method traverses the unspliced ​​FilterOperator binary tree to determine whether the node is a leaf node. If so, the path of the leaf node is taken. If the path starts with time or root, it is not processed, otherwise the device name and node are not processed. The paths are spliced ​​and returned; if not, all children of the node are iteratively processed. In the example, the result of splicing the filter conditions of device 1 is `time = 1 AND root.sg.d1.s1 <25`, and device 2 is` time = 1 AND root.sg.d2.s1 <25`.

The following example summarizes the variable information calculated through this stage:

-measurement list `measurements`:` [s1, "1", s1, s2, s2, s5] `
-measurement type `measurementTypeMap`:
  -`s1-> Exist`
  -`s2-> Exist`
  -`" 1 "-> Constant`
  -`s5-> NonExist`
-DeviceToMeasurementsMap for each device:
  -`root.sg.d1-> s1, s2`
  -`root.sg.d2-> s1`
-Filter condition `deviceToFilterMap` for each device:
  -`root.sg.d1-> time = 1 AND root.sg.d1.s1 <25`
  -`root.sg.d2-> time = 1 AND root.sg.d2.s1 <25`

### Constructing a Table Header (ColumnHeader)

-org.apache.iotdb.db.service.TSServiceImpl

After generating the physical plan, you can execute the executeQueryStatement () method in TSServiceImpl to generate a result set and return it. The first step is to construct the table header.

Query by device alignment After calling the TSServiceImpl.getQueryColumnHeaders () method, enter TSServiceImpl.getAlignByDeviceQueryHeaders () according to the query type to construct the headers.

The `getAlignByDeviceQueryHeaders ()` method is declared as follows:

```java
private void getAlignByDeviceQueryHeaders (
      AlignByDevicePlan plan, List <String> respColumns, List <String> columnTypes)
Input: the currently executing physical plan AlignByDevicePlan and the column names respColumns and their corresponding data type columnTypes
Output: Calculated column name respColumns and data type columnTypes
```

The specific implementation logic is as follows:

1. First add the `Device` column, whose data type is` TEXT`;
2. Traverse the list of unduplicated measurements to determine the type of measurement currently being traversed. If it is an Exist type, obtain its type from the measurementTypeMap. For the other two types, set its type to TEXT. Types are added to the header data structure.
3. Deduplicate measurements based on the intermediate variable `deduplicatedMeasurements`.

The resulting header is:

Time | Device | s1 | 1 | s1 | s2 | s2 | s5 |
| ---- | ------ | --- | --- | --- | --- | --- | --- |
| | | | | | | | |

The de-duplicated `measurements` is` [s1, "1", s2, s5] `.

### Result Set Generation

After the ColumnHeader is generated, the final step is to populate the result set with the results and return.

#### Result Set Creation

-org.apache.iotdb.db.service.TSServiceImpl

At this stage, you need to call `TSServiceImpl.createQueryDataSet ()` to create a new result set. This part of the implementation logic is relatively simple. For AlignByDeviceQuery, you only need to create a new `AlignByDeviceDataSet`. The parameters in AlignByDevicePlan will be set in the constructor. Assign to the newly created result set.

#### Result Set Fill

-org.apache.iotdb.db.utils.QueryDataSetUtils

Next you need to fill the results. AlignByDeviceQuery will call the `TSServiceImpl.fillRpcReturnData ()` method, and then enter the `QueryDataSetUtils.convertQueryDataSetByFetchSize ()` method according to the query type.

The important method for getting results in the `convertQueryDataSetByFetchSize ()` method is the `hasNext ()` method of QueryDataSet.

The main logic of the `hasNext ()` method is as follows:

1. Determine if there is a specified row offset `rowOffset`, and if so, skip the number of rows to be offset; if the total number of results is less than the specified offset, return false.
2. Determine if there is a specified limit on the number of rows `rowLimit`, if there is, compare the current number of output lines, and return false if the current number of output lines is greater than the limit.
3. Enter `AlignByDeviceDataSet.hasNextWithoutConstraint ()` method

<br>

-org.apache.iotdb.db.query.dataset.AlignByDeviceDataSet

First explain the meaning of the important fields in the result set:

-`deviceIterator`: The device-by-device query essentially calculates the mapping value and filter conditions corresponding to each device, and then performs the query separately for each device. This field is an iterator for the device, and each query gets a device to perform.
-`currentDataSet`: This field represents the result set obtained by this query on a device.

The work done by the `hasNextWithoutConstraint ()` method is mainly to determine whether the current result set has the next result, if not, the next device is obtained, the path, data type and filter conditions required by the device to execute the query are calculated, and then executed according to its query type The result set is obtained after a specific query plan, until no device is available for querying.

The specific implementation logic is as follows:

1. First determine whether the current result set is initialized and there is a next result. If it is, it will return true directly, that is, you can currently call the `next ()` method to get the next `RowRecord`; otherwise, the result set is not initialized and proceed to step 2. .
2. Iterate `deviceIterator` to get the devices needed for this execution, and then get the corresponding measurement points for that device in` deviceToMeasurementsMap` to get `executeColumns`.
3. Concatenate the current device name and measurements to calculate the query path, data type, and filter conditions of the current device. The corresponding fields are `executePaths`,` tsDataTypes`, and `expression`. If it is an aggregate query, you need to calculate` executeAggregations`.
4. Determine whether the current subquery type is GroupByQuery, AggregationQuery, FillQuery or RawDataQuery. Perform the corresponding query and return the result set. For the implementation logic, refer to [Raw Data Query] (/ # / SystemDesign / progress / chap5 / sec3), [Aggregation Query] (/ # / SystemDesign / progress / chap5 / sec4), [downsampling query] (/ # / SystemDesign / progress / chap5 / sec5).

After initializing the result set through the `hasNextWithoutConstraint ()` method and ensuring that there is a next result, you can call the `QueryDataSet.next ()` method to get the next `RowRecord`.

The `next ()` method is mainly implemented as the `AlignByDeviceDataSet.nextWithoutConstraint ()` method.

The work done by the `nextWithoutConstraint ()` method is to ** transform the time-aligned result set form obtained by a single device query into a device-aligned result set form **, and return the transformed `RowRecord`.

The specific implementation logic is as follows:

1. First get the next time-aligned `originRowRecord` from the result set.
2. Create a new `RowRecord` with timestamp, add device columns to it, first create a Map structure` currentColumnMap` of `measurementName-> Field` according to` executeColumns` and the obtained result.
3. It only needs to traverse the deduplicated `measurements` list to determine its type. If the type is` Exist`, then obtain the corresponding result from the `currentColumnMap` according to the measurementName. If not, set it to` null`; if it is `NonExist` type, it is set to` null` directly; if it is `Constant` type,` measureName` is used as the value of this column.

After writing the output data stream according to the transformed `RowRecord`, the result set can be returned.
