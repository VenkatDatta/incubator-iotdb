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

# data query

There are several types of data queries

* Query of raw data
* Aggregation query
* Downsampling query
* Single point supplementary value query
* Latest data query

In order to achieve the above kinds of queries, a basic query component for a single time series is designed in the IoTDB query engine. On this basis, a variety of query functions are implemented.

## Related documents

* [Basic Query Component] (/ # / SystemDesign / progress / chap5 / sec2)
* [Raw data query] (/ # / SystemDesign / progress / chap5 / sec3)
* [Aggregation query] (/ # / SystemDesign / progress / chap5 / sec4)
* [Downsampling query] (/ # / SystemDesign / progress / chap5 / sec5)
* [Recent Timestamp Query] (/ # / SystemDesign / progress / chap5 / sec6)