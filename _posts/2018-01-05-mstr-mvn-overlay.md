---
layout: post
title: Maven WAR overlay for MSTR Web customizations
---

MicroStrategy doesn't provide a lot of guidance on how to manage Web plugins in source control, or build and deployment automation.  However, it turns out that [Maven WAR overlays](https://maven.apache.org/plugins/maven-war-plugin/overlays.html) are the perfect solution to MSTR web customizations. 

WAR overlays are specifically designed to solve the problem of applying customizations to add or override specific files within an off-the-shelf base webapp (packaged as WAR).  Other customizable webapps like [CAS (for single sign-on) recommend a similar approach](https://apereo.github.io/cas/5.0.x/installation/Maven-Overlay-Installation.html).  

Using Maven WAR overlays, we can accomplish the following goals

* Only the actual customizations are checked in.  The base webapp contents are not. 
* Any developer can check out the MSTR web customizations and get a development environment running in minutes.
* The customized MSTR Web can be built, tested and deployed through standard CI tools like Bamboo or Jenkins.
* Developers can work on code for web customizations in an IDE (e.g., Eclipse) including deployment to a local app server (e.g., Tomcat) for testing.
* Customizations are applied to upgrades by changing the version of the provided dependency
* The MSTR Web Customization Editor (Eclipse plugin) can still be used to provide assistance with understanding what options are available for pages, templates etc.

The Maven setup itself is straightforward.  "mvn package" and "mvn install" will work as with any other webapp project.  You can import this project into Eclipse with M2E, and Eclipse WebTools (WTP) will work as well.

Example pom.xml fragment:

```
    <properties>
        <mstr.version>10.9</mstr.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>com.microstrategy</groupId>
            <artifactId>web</artifactId>
            <version>${mstr.version}</version>
            <scope>runtime</scope>
            <type>war</type>
        </dependency>
        <dependency>
            <groupId>com.microstrategy</groupId>
            <artifactId>WebApp</artifactId>
            <version>${mstr.version}</version>
            <scope>provided</scope>
        </dependency>
        <!-- repeat for WebBeans, WebObjects etc. -->
    </dependencies>
    <build>
        <finalName>customized-mstr-web</finalName>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-war-plugin</artifactId>
                <configuration>
                    <overlays>
                        <overlay>
                            <groupId>com.microstrategy</groupId>
                            <artifactId>web</artifactId>
                        </overlay>
                    </overlays>
                    <failOnMissingWebXml>false</failOnMissingWebXml>
                </configuration>
            </plugin>
        </plugins>
    </build>
```

The only thing that makes MSTR unique relative to other WAR overlay use cases is that MSTR does not provide any public repository with its webapp or libraries.  So you have to install them yourself in a private or [local repository](https://stackoverflow.com/questions/1164043/maven-how-to-include-jars-which-are-not-available-in-reps-into-a-j2ee-project).  Additionally you have to install the JARs within the WAR as artifacts in that same repository, so that any custom Java classes can resolve dependencies to the MSTR libraries.  Those are referenced in the POM with scope `provided` as they will be present at runtime from the parent WAR and only need to be available for compile and testing.

Finally, you can still use the Web Customization Editor (WCE) in Eclipse but it takes some manual effort. The WCE expects to point to a fully-expanded MSTR webapp, which makes it hard to work with source control and CI processes.  The workaround I used was to treat MSTR Web as a normal Maven/Eclipse project, and then point the WCE at the Eclipse WTP webapps folder. The path for that will look something like 
`${workspace}\.metadata\.plugins\org.eclipse.wst.server.core\tmp0\wtpwebapps\${project}` where `${workspace}` is the path to your Eclipse workspace and `${project}` is your project name in Eclipse.  You can then work with the MicroStrategy perspective and publish your changes to your running MSTR Web instantly.  But when you want to check your changes in, you will have to manually copy any files from your live Tomcat plugin folder to your primary project's working copy.