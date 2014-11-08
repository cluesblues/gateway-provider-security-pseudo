<!---
   Licensed to the Apache Software Foundation (ASF) under one or more
   contributor license agreements.  See the NOTICE file distributed with
   this work for additional information regarding copyright ownership.
   The ASF licenses this file to You under the Apache License, Version 2.0
   (the "License"); you may not use this file except in compliance with
   the License.  You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
--->
gateway-provider-security-pseudo
===================

Companion project for blog on adding a simple federation provider to Apache Knox

[Article](AddingFederationProvider.md)

Clone or fork this project and build it with:

mvn clean install

You can then add it to the a Knox deployment by adding the resulting jar file to the {GATEWAY_HOME}/ext directory.
It will be picked by being on the classpath and discovered using the ServiceLoader API for finding certain interface implementations.


