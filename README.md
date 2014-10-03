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

You may clone this project into your project for Apache Knox at the root level and it will add the module.
Once it is cloned, you will need to delete the git specific files that are added in order for the build to not choke on the fact that they are missing apache license headers.

    rm gateway-provider-security-pseudo/.git/config
    rm gateway-provider-security-pseudo/.git/description
    rm gateway-provider-security-pseudo/.git/HEAD
    rm gateway-provider-security-pseudo/.git/hooks/applypatch-msg.sample
    rm gateway-provider-security-pseudo/.git/hooks/commit-msg.sample
    rm gateway-provider-security-pseudo/.git/hooks/post-update.sample
    rm gateway-provider-security-pseudo/.git/hooks/pre-applypatch.sample
    rm gateway-provider-security-pseudo/.git/hooks/pre-commit.sample
    rm gateway-provider-security-pseudo/.git/hooks/pre-push.sample
    rm gateway-provider-security-pseudo/.git/hooks/pre-rebase.sample
    rm gateway-provider-security-pseudo/.git/hooks/prepare-commit-msg.sample
    rm gateway-provider-security-pseudo/.git/hooks/update.sample
    rm gateway-provider-security-pseudo/.git/info/exclude
    rm gateway-provider-security-pseudo/.git/logs/HEAD
    rm gateway-provider-security-pseudo/.git/logs/refs/heads/master
    rm gateway-provider-security-pseudo/.git/logs/refs/remotes/origin/HEAD
    rm gateway-provider-security-pseudo/.git/packed-refs
    rm gateway-provider-security-pseudo/.git/refs/heads/master
    rm gateway-provider-security-pseudo/.git/refs/remotes/origin/HEAD

You will also want to delete the markdown files for the same reason:

    rm gateway-provider-security-pseudo/*.md

Then follow the instructions in the article to tie the new module into the gateway project and release module pom.xml files.

