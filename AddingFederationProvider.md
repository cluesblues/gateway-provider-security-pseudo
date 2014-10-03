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

# Adding a Federation Provider to Apache Knox

## Pseudo Federation Provider
This article will walk through the process of adding a new provider for establishing the identity of a user. The simple example of the Pseudo authentication mechanism in Hadoop will be used to communicate the general ideas for extending the preauthenticated federation provider that is available out of the box in Apache Knox. This is not a provider that should be used in a production environment and has at least one major limitation. It will however illustrate the general programming model for adding preauthenticated federation providers.

### Provider Types
Apache Knox has two types of providers for establishing the identity of the source of an incoming REST request. One is an Authentication Provider and the other is a Federation Provider.

### Authentication Providers
Authentication providers are responsible for actually collecting credentials of some sort from the end user. Examples of authentication providers would be things like HTTP BASIC authentication with username and password that gets authenticated against LDAP or RDBMS, etc. Apache Knox ships with HTTP BASIC authentication against LDAP using Apache Shiro. The Shiro provider can actually be configured in multiple ways.

Authentication providers are sometimes less than ideal since many organizations only want their users to provide credentials to the enterprise trusted/preferred solutions and to use some sort of SSO or federation of that authentication event across all other applications.

### Federation Providers
Federation providers, on the other hand, never see the users' actual credentials but instead federated a previous authentication event through the processing and validation of some sort of token. This allows for greater isolation and protection of user credentials while still providing some means to verify the trustworthiness of the incoming identity assertions. Examples of federation providers would be things like OAuth 2, SAML Assertions, JWT/SWT tokens, Header based identity propagation, etc. Out of the box, Apache Knox enables the use of custom headers for propagating things like the user principal and group membership through the HeaderPreAuth federation provider.

This is generally useful, for solutions such as CA SiteMinder and IBM Tivoli Access Manager. In these sorts of deployments, all traffic to Hadoop would have to be go through the solution gateway which authenticates the user and can inject the request with identity propagation headers. The fact that the network security does not allow for requests to bypass the solution gateway provides a level of trust for accepting the header based identity assertions. We also provide for additional validation through a pluggable mechanism and have an ip address validation that can also be used.

### Let's add a Federation Provider
This article will discuss what is involved in adding a new federation provider that will actually extend the abstract bases that were introduced in the PreAuth provider module. It will be a very minimal provider that accepts a request parameter from the incoming request as the user's principal.

### The module and dependencies
The Apache Knox project uses Apache Maven for build and dependency management. We will need to create a new module for the Pseudo federation provider and include our own pom.xml.

    <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.apache.knox</groupId>
        <artifactId>gateway</artifactId>
        <version>0.6.0-SNAPSHOT</version>
    </parent>
    <artifactId>gateway-provider-security-pseudo</artifactId>

    <name>gateway-provider-security-pseudo</name>
    <description>An extension of the gateway introducing support for user.name request parameters.</description>

    <licenses>
        <license>
            <name>The Apache Software License, Version 2.0</name>
            <url>http://www.apache.org/licenses/LICENSE-2.0.txt</url>
            <distribution>repo</distribution>
        </license>
    </licenses>

    <dependencies>
        <dependency>
            <groupId>org.apache.knox</groupId>
            <artifactId>gateway-provider-security-preauth</artifactId>
        </dependency>

        <dependency>
            <groupId>org.apache.knox</groupId>
            <artifactId>gateway-spi</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.knox</groupId>
            <artifactId>gateway-util-common</artifactId>
        </dependency>

        <dependency>
            <groupId>org.eclipse.jetty.orbit</groupId>
            <artifactId>javax.servlet</artifactId>
        </dependency>

        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.easymock</groupId>
            <artifactId>easymock</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.apache.knox</groupId>
            <artifactId>gateway-test-utils</artifactId>
            <scope>test</scope>
        </dependency>

    </dependencies>
    </project>

#### Dependencies

NOTE: the "version" element must match the version indicated in the pom.xml of the Knox project. Otherwise, building will fail.

##### gateway-provider-security-preauth
This particular federation provider is going to extend the existing PreAuth module with the capability to accept the user.name request parameter as an assertion of the identity by a trusted party. Therefore, we will depend on the preauth module in order to leverage the facilities available in the base classes available there for things like ip address validation, etc.

##### gateway-spi
The gateway-spi dependency above pulls in the general interfaces, base classes and utilities that are expected for extended the Apache Knox gateway. The core GatewayServices are available through the gateway-spi module as well as a number of other foundational elements of gateway development.

##### gateway-util-common
This gateway-util-common module, as the name suggests, provides common utility facilities for the developing the gateway product. This is where you find the auditing, JSON and url utilities classes for gateway development.

##### javax.servlet from org.eclipse.jetty.orbit
This module provides the servlet filter specific classes that are need for the provider filter implementation.

##### junit, easymock and gateway-test-utils
JUnit, easymock and gateway-test-utils provide the basis for writing REST based unit tests for the Apache Knox Gateway project and can be found in all of the existing unit tests for the various modules that make up the gateway offering.

### Apache Knox Topologies
In Apache Knox, individual Hadoop clusters are represented by descriptors called topologies that result in the deployment of specific endpoints that expose and protect access to the services of the associated Hadoop cluster. The topology descriptor describes the available services and their respective URL's within the actual Hadoop cluster as well as the policy for protecting access to those services. The policy is defined through the description of various Providers. Each provider and service within a Knox topology has a role and provider roles consist of: authentication, federation, authorization, identity assertion, etc. In this article we are concerned with a provider of type federation.

Since the Pseudo provider is assuming that authentication has happened at the OS level or from within another piece of middleware and that credentials were exchanged with some party other than Knox, we will be making this a federation provider. The typical provider configuration will look something like this:

    <provider>
      <role>federation</role>
      <name>Pseudo</name>
      <enabled>true</enabled>
    </provider>

Ultimately, an Apache Knox topology manifests as a web application deployed within the gateway process that exposes and protects the URLs associated with the services of the underlying Hadoop components in each cluster. Providers generally interject a ServletFilter into the processing path of the REST API requests that enter the gateway and are dispatched to the Hadoop cluster. The mechanism used to interject the filters, their related configuration and integration into the gateway is the ProviderDeploymentContributor.

### ProviderDeploymentContributor
    /**
    * Licensed to the Apache Software Foundation (ASF) under one
    * or more contributor license agreements.  See the NOTICE file
    * distributed with this work for additional information
    * regarding copyright ownership.  The ASF licenses this file
    * to you under the Apache License, Version 2.0 (the
    * "License"); you may not use this file except in compliance
    * with the License.  You may obtain a copy of the License at
    *
    *     http://www.apache.org/licenses/LICENSE-2.0
    *
    * Unless required by applicable law or agreed to in writing, software
    * distributed under the License is distributed on an "AS IS" BASIS,
    * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    * See the License for the specific language governing permissions and
    * limitations under the License.
    */
    package org.apache.hadoop.gateway.preauth.deploy;

    import java.util.ArrayList;
    import java.util.List;
    import java.util.Map;
    import java.util.Map.Entry;
    
    import org.apache.hadoop.gateway.deploy.DeploymentContext;
    import org.apache.hadoop.gateway.deploy.ProviderDeploymentContributorBase;
    import org.apache.hadoop.gateway.descriptor.FilterParamDescriptor;
    import org.apache.hadoop.gateway.descriptor.ResourceDescriptor;
    import org.apache.hadoop.gateway.topology.Provider;
    import org.apache.hadoop.gateway.topology.Service;
    
    public class PseudoAuthContributor extends
        ProviderDeploymentContributorBase {
      private static final String ROLE = "federation";
      private static final String NAME = "Pseudo";
      private static final String PREAUTH_FILTER_CLASSNAME = "org.apache.hadoop.gateway.preauth.filter.PseudoAuthFederationFilter";
      
      @Override
      public String getRole() {
        return ROLE;
      }
      
      @Override
      public String getName() {
        return NAME;
      }
      
      @Override
      public void contributeFilter(DeploymentContext context, Provider provider, Service service, 
          ResourceDescriptor resource, List<FilterParamDescriptor> params) {
        // blindly add all the provider params as filter init params
        if (params == null) {
          params = new ArrayList<FilterParamDescriptor>();
        }
        Map<String, String> providerParams = provider.getParams();
        for(Entry<String, String> entry : providerParams.entrySet()) {
          params.add( resource.createFilterParam().name( entry.getKey().toLowerCase() ).value( entry.getValue() ) );
        }
        resource.addFilter().name( getName() ).role( getRole() ).impl( PREAUTH_FILTER_CLASSNAME ).params( params );
      }
    }

The way in which the required DeploymentContributors for a given topology are located is based on the use of the role and the name of the provider as indicated within the topology descriptor. The topology deployment machinery within Knox first looks up the requried DeploymentContributor by role. In this case, it identifies the identity provider as being a type of federation. It then looks for the federation provider with the name of Pseudo.

Once the providers have been resolved into the required set of DeploymentContributors each contributor is given the opportunity to *contribute* to the construction of the topology web application that exposes and protects the service APIs within the Hadoop cluster.

This particular DeploymentContributor needs to add the PseudoAuthFederationFilter servlet Filter implementation to the topology specific filter chain. In addition to adding the filter to the chain, this provider will also add each of the provider params from the topology descriptor as filterConfig parameters. This enables the configuration of the resulting servlet filters from within the topology descriptor while enacapsulating the specific implementation details of the provider from the end user.

### PseudoAuthFederationFilter
    /**
     * Licensed to the Apache Software Foundation (ASF) under one
     * or more contributor license agreements.  See the NOTICE file
     * distributed with this work for additional information
     * regarding copyright ownership.  The ASF licenses this file
     * to you under the Apache License, Version 2.0 (the
     * "License"); you may not use this file except in compliance
     * with the License.  You may obtain a copy of the License at
     *
     *     http://www.apache.org/licenses/LICENSE-2.0
     *
     * Unless required by applicable law or agreed to in writing, software
     * distributed under the License is distributed on an "AS IS" BASIS,
     * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
     * See the License for the specific language governing permissions and
     * limitations under the License.
     */
    package org.apache.hadoop.gateway.preauth.filter;
    
    import java.security.Principal;
    import java.util.Set;
    
    import javax.servlet.FilterConfig;
    import javax.servlet.ServletException;
    import javax.servlet.http.HttpServletRequest;
    
    public class PseudoAuthFederationFilter 
      extends AbstractPreAuthFederationFilter {
      
      @Override
      public void init(FilterConfig filterConfig) throws ServletException {
        super.init(filterConfig);
      }
    
      /**
       * @param httpRequest
       */
      @Override
      protected String getPrimaryPrincipal(HttpServletRequest httpRequest) {
        return httpRequest.getParameter("user.name");
      }
    
      /**
       * @param principals
       */
      @Override
      protected void addGroupPrincipals(HttpServletRequest request, 
          Set<Principal> principals) {
        // pseudo auth currently has no assertion of group membership
      }
    }

The PseudoAuthFederationFilter above extends AbstractPreAuthFederationFilter. This particular base class takes care of a number of boilerplate type aspects of preauthenticated providers that would otherwise have to be done redundantly across providers. The two abstract methods that are specific to each provider are getPrimaryPrincipal and addGroupPrincipals. These methods are called by the base class in order to determine what principals should be created and added to the java Subject that will become the effective user identity for the request processing of the incoming request.

#### getPrimaryPrincipal
Implementing the abstract method getPrimaryPrincipal allows the new provider to extract the established identity from the incoming request or however appropriate for the given provider and communicate it back the the AbstractPreAuthFederationFilter which will in turn add it to the java Subject being created to represent the user's identity. For this particular provider, all we have to do is return the request parameter by the name of "user.name".

#### addGroupPrincipals
Given a set of Principals, the addGroupPrincipals is an opportunity to add additional group principals to the resulting java Subject that will be used to represent the user's identity. This is specifically done by adding new org.apache.hadoop.gateway.security.GroupPrincipals to the set. For the Pseudo authentication mechanism in Hadoop, there really is no way to communicate the group membership through the request parameters. One could easily envision adding an additional request parameter for this though - something like "user.groups".

### Configure as an Available Provider
In order for the deployment machinery to be able to discover the availability of your new provider implementation, you will need to make sure that the org.apache.hadoop.gateway.deploy.ProviderDeploymentContributor file is in the resources/META-INF/services directory and that it contains the classname of the new provider's DeploymentContributor - in this case PseudoAuthContributor.

#### resources/META-INF/services/org.apache.hadoop.gateway.deploy.ProviderDeploymentContributor
    ##########################################################################
    # Licensed to the Apache Software Foundation (ASF) under one
    # or more contributor license agreements.  See the NOTICE file
    # distributed with this work for additional information
    # regarding copyright ownership.  The ASF licenses this file
    # to you under the Apache License, Version 2.0 (the
    # "License"); you may not use this file except in compliance
    # with the License.  You may obtain a copy of the License at
    #
    #     http://www.apache.org/licenses/LICENSE-2.0
    #
    # Unless required by applicable law or agreed to in writing, software
    # distributed under the License is distributed on an "AS IS" BASIS,
    # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    # See the License for the specific language governing permissions and
    # limitations under the License.
    ##########################################################################
    
    org.apache.hadoop.gateway.preauth.deploy.PseudoAuthContributor

### Add to Knox as a Gateway Module
At this point, the module should be able to be built as a standalone module with:

    mvn clean install

However, we want to extend the Apache Knox Gateway build to include the new module in its build and release processes. In order to do this we will need to add it to a common pom.xml files.

#### Root Level Pom.xml
At the root of the project source tree there is a pom.xml file that defines all of the modules that are official components of the gateway server release. You can find each of these modules in the "modules" element. We need to add our new module declaration there:
    
    <modules>
      ...
      <module>gateway-provider-security-pseudo</module>
      ...
    </modules>
    
Then later in the same file we have to add a fuller definition of our module to the dependencyManagement/dependencies element:

    <dependencyManagement>
        <dependencies>
            ...
            <dependency>
                <groupId>${gateway-group}</groupId>
                <artifactId>gateway-provider-security-pseudo</artifactId>
                <version>${gateway-version}</version>
            </dependency>
            ...
        </dependencies>
    </dependencyManagement>

#### Gateway Release Module Pom.xml
Now, our Pseudo federation provider is building with the gateway project but it isn't quite included in the gateway server release artifacts. In order to get it included in the release archives and available to the runtime, we need to add it as a dependency to the appropriate release module. In this case, we are adding it to the pom.xml file within the gateway-release module:

    <dependencies>
        ...
        <dependency>
            <groupId>${gateway-group}</groupId>
            <artifactId>gateway-provider-security-pseudo</artifactId>
        </dependency>
        ...
    </dependencies>
    
Note that this is basically the same definition that was added to the root level pom.xml minus the "version" element.

### Build, Test and Deploy
At this point, we should have an integrated custom component that can be described for use within the Apache Knox topology descriptor file and engaged in the authentication of incoming requests for resources of the protected Hadoop cluster.

#### building
You may use the same maven commands to:

    mvn clean install
    
This will build and run the gateway unit tests.

You may also use the following to not only build and run the tests but to also package up the release artifacts. This is a great way to quickly setup a test instance in order to manually test your new Knox functionality.

    ant package

#### testing
To install the newly packaged release archive in a GATEWAY_HOME environment:

    ant install-test-home
    
This will unzip the release bits into a local ./install directory and do some initial setup tasks to ensure that it is actually runnable.

We can now start a test ldap server that is seeded with a couple test users:

    ant start-test-ldap

The sample topology files are setup to authenticate against this LDAP server for convenience and can be used as is in order to quickly do a sanity test of the install.

At this point, we can choose to run a test Knox instance or a debug Knox instance. If you want to run a test instance without the ability to connect a debugger then:

    ant start-test-gateway

If you would like to connect a debugger and step through the code to debug or ensure that your functionality is running as expected then you need a debug instance:

    ant start-debug-gateway
    
#### curl
You may now test the out of the box authentication against LDAP using HTTP BASIC by using curl and one of the simpler APIs exposed by Apache Knox:

    curl -ivk --user guest:guest-password "https://localhost:8443/gateway/sandbox/webhdfs/v1/tmp?op=LISTSTATUS"

#### Change Topology Descriptor
Once the server is up and running and you are able to authenticate with HTTP BASIC against the test LDAP server, you can now change the topology descriptor to leverage your new federation provider.

Find the sandbox.xml file in the install/conf/topologies file and edit it to reflect your provider type, name and any provider specific parameters.

    <provider>
       <role>federation</role>
       <name>PseudoProvider</name>
       <enabled>true</enabled>
       <param>
           <name>filter-init-param-name</name>
           <value>value</value>
       </param>
    </provider

Once your federation provider is configured, just save the topology descriptor. Apache Knox will notice that the file has changed and automatically redeploy that particular topology. Any provider params described in the provider element will be added to the PseudoAuthFederationFilter as servlet filter init params and can be used to configure aspects of the filter's behavior.

#### curl again
We are now ready to use curl again to test the new federation provider and ensure that it is working as expected:

    curl -ivk "https://localhost:8443/gateway/sandbox/webhdfs/v1/tmp?op=LISTSTATUS&user.name=guest"

## More Resources
Apache Knox Developers Guide: http://knox.apache.org/books/knox-0-4-0/dev-guide.html

Apache Knox Users Guide: http://knox.apache.org/books/knox-0-4-0/knox-0-4-0.html

Github project for this article: https://github.com/lmccay/gateway-provider-security-pseudo

## Conclusion
This article has illustrated a simplified example of implementing a federation provider for establishing the identity of a previous authentication event and propagating that into the request processing for Hadoop REST APIs inside of Apache Knox. 

The process to extend the preauthenticated federation provider is a quick and simple way to extend certain SSO capabilities into providing authenticated access to Hadoop resources through Apache Knox.

The Knox community is a growing community that welcomes contributions from interested users in order to grow the capabilities to include truly useful and impactful features.

NOTE: It is important to understand that the provider illustrated in this example has limitations that preclude it from being used in production. Most notably, it does not have any means to follow redirects due to the missing user.name parameter in the Location header. In order to do this, we would need to set a cookie to determine the user identity on the redirected request.
