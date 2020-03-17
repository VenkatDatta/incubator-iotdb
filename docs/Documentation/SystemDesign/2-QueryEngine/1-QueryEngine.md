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

# Query Engine

<img style = "width: 100%; max-width: 800px; max-height: 600px; margin-left: auto; margin-right: auto; display: block;" src = "https: // user-images. githubusercontent.com/19167280/73625242-f648a100-467e-11ea-921c-b954a3ecae7a.png ">

## Design thinking

The query engine is responsible for parsing all user commands, generating plans, delivering them to the corresponding executors, and returning result sets.

## Related classes

* org.apache.iotdb.db.service.TSServiceImpl

IoTDB server-side RPC implementation, which directly interacts with the client.
Ranch
* org.apache.iotdb.db.qp.Planner
Ranch
Parse SQL, generate logical plans, optimize logic, and generate physical plans.

* org.apache.iotdb.db.qp.executor.PlanExecutor

Distribute the physical plan to the corresponding actuators, including the following four specific actuators.
Ranch
* MManager: metadata operations
* StorageEngine: data write
* QueryRouter: data query
* LocalFileAuthorizer: Authorization operation

* org.apache.iotdb.db.query.dataset. *

The batch result set is returned to the client and contains part of the query logic.

## Query Process

* SQL parsing
* Generate logical plan
* Generate physical plan
* Construct Result Set Generator
* Return the result set in batches

## Related documents

* [Query plan generator] (/ # / SystemDesign / progress / chap2 / sec2)
* [Plan Executor] (/ # / SystemDesign / progress / chap2 / sec3)