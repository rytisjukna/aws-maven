# AWS Maven Wagon

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)


## Description
This project is a fork of a [Maven Wagon](https://github.com/spring-projects/aws-maven) for [Amazon S3](http://aws.amazon.com/s3/).  
In order to to publish artifacts to an S3 bucket, the user (as identified by their access key) must have several permissions on the S3 bucket (PutObject, PutObjectAcl, GetObject).


## Why this fork?
- if you want to deploy to the limited-access S3 buckets (no public access, no ownership).


## Usage

### Setting up S3 bucket and permissions

Create a bucket on your S3: note the region, block public access to it, and set a policy.
Here's a policy that worked for me:

```json
{
    "Version": "2008-10-17",
    "Id": "Policy<SOMENUMBERHERE>",
    "Statement": [
        {
            "Sid": "<SOMEIDHERE>",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<XXXXXX>:user/<USERNAME>"
            },
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::<BUCKET>"
        },
        {
            "Sid": "<SOMEOTHERIDHERE>",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<XXXXXXX>:user/<USERNAME>"
            },
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:PutObjectAcl"
            ],
            "Resource": "arn:aws:s3:::<BUCKET>/*"
        }
    ]
}
```

Make sure the user listed above (arn:aws:iam....) has at least these permissions (IAM -> Policies):
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "<WHATEVER>",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:PutObjectAcl",
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::<BUCKET>/*",
                "arn:aws:s3:::<BUCKET>"
            ]
        }
    ]
}
```

### Setting up your AWS credentials in your local machine

Look at your `~/.aws/` folder, create one if missing.
A file `config` must contain at least:
```bash
[default]
region=<YOUR BUCKET REGION ID, FOR INSTANCE eu-central-1>
```

A file `credentials` must contain:
```bash
[default]
aws_access_key_id=<ACCESS KEY OF YOUR USER>
aws_secret_access_key=<SECRET ACCESS KEY OF YOUR USER>
```

Alternatively, the access and secret keys for the account can be provided using (applied in order below)

* `aws.accessKeyId` and `aws.secretKey` [system properties](http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/auth/SystemPropertiesCredentialsProvider.html)
* `AWS_ACCESS_KEY_ID` (or `AWS_ACCESS_KEY`) and `AWS_SECRET_KEY` (or `AWS_SECRET_ACCESS_KEY`) [environment variables](http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/auth/EnvironmentVariableCredentialsProvider.html)
* The Amazon EC2 [Instance Metadata Service](https://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/auth/EC2ContainerCredentialsProviderWrapper.html)


### Preparing Maven to recognize S3 protocol

The `~/.m2/settings.xml` must be updated to reference S3 Wagon:

```xml
<settings>
  ...
  <servers>
    ...
    <server>
      <id>aws-release</id>
      <configuration>
        <wagonProvider>s3</wagonProvider>
      </configuration>
    </server>
    <server>
      <id>aws-snapshot</id>
      <configuration>
        <wagonProvider>s3</wagonProvider>
      </configuration>
    </server>
    ...
  </servers>
  ...
</settings>
```

### Publishing Maven artifacts
A S3 build extension must be defined in your maven *artifact*'s `pom.xml`. 

The latest version of the wagon can be found at http://rytisjunka.github.io

```xml
<project>
  ...
<pluginRepositories>
    ...
    <!-- public github repository, hosts a proper aws s3 plugin for private repos -->
	<pluginRepository>
      <id>rytisjukna-plugins</id>
      <name>Rytis Jukna plugins</name>
      <url>https://rytisjukna.github.io/artifacts</url>

      <releases>
        <enabled>true</enabled>
        <updatePolicy>never</updatePolicy>
      </releases>
      <snapshots>
        <enabled>false</enabled>
        <updatePolicy>never</updatePolicy>
      </snapshots>
      <layout>default</layout>
    </pluginRepository>
    ...
</pluginRepositories>

  ...
  <build>
    ...
    <extensions>
      ...
      <extension>
        <!-- allows using private S3 bucket for Maven artifact repo -->
        <groupId>com.github.rytisjukna</groupId>
        <artifactId>aws-maven</artifactId>
        <version>6.0.1</version>
      </extension>
      ...
    </extensions>
    ...
  </build>
  ...
</project>
```

Once the build extension is configured, the distribution management repositories can be defined in the maven *artifact*'s `pom.xml` with an `s3://` scheme.

```xml
<project>
  ...
  <distributionManagement>
    <!-- our private secret repo for Maven artifacts -->
    <repository>
      <id>aws-release</id>
      <name>AWS Release Repository</name>
      <url>s3://<BUCKET>/release</url>
    </repository>
    <snapshotRepository>
      <id>aws-snapshot</id>
      <name>AWS Snapshot Repository</name>
      <url>s3://<BUCKET>/snapshot</url>
    </snapshotRepository>
  </distributionManagement>
  ...
</project>
```

You publish your artifacts to S3 bucket by simple 
```bash
mvn deploy
```

### Consuming Maven artifacts
Your *project* `pom.xml` file should refer to the public Github repo for the plugin, and for the private artifact repo on S3:
```xml
...
<pluginRepositories>
    ...
    <!-- public github repository, hosts a proper aws s3 plugin for private repos -->
	<pluginRepository>
      <id>rytisjukna-plugins</id>
      <url>https://rytisjukna.github.io/artifacts</url>
      <releases>
        <enabled>true</enabled>
        <updatePolicy>never</updatePolicy>
      </releases>
      <snapshots>
        <enabled>false</enabled>
        <updatePolicy>never</updatePolicy>
      </snapshots>
      <layout>default</layout>
    </pluginRepository>
    ...
</pluginRepositories>
<repositories>
    <!-- our private secret repo for Maven artifacts -->
     <repository>
      <id>aws-release</id>
      <name>our private AWS Release Repository</name>
      <url>s3://<BUCKET>/release</url>
    </repository>
</repositories>
  ...
  <build>
    ...
    <extensions>
      ...
      <extension>
        <!-- allows using private S3 bucket for Maven artifact repo -->
        <groupId>com.github.rytisjukna</groupId>
        <artifactId>aws-maven</artifactId>
        <version>6.0.1</version>
      </extension>
      ...
    </extensions>
    ...
  </build>

```


### Connecting through a Proxy
For being able to connect behind an HTTP proxy you need to add the following configuration to `~/.m2/settings.xml`:

```xml
<settings>
  ...
  <proxies>
     ...
     <proxy>
         <active>true</active>
         <protocol>s3</protocol>
         <host>myproxy.host.com</host>
         <port>8080</port>
         <username>proxyuser</username>
         <password>somepassword</password>
         <nonProxyHosts>www.google.com|*.somewhere.com</nonProxyHosts>
     </proxy>
     ...
    </proxies>
  ...
</settings>
```

## Release Notes
* `6.0.2`
   - upgraded to java 11

* `6.0.1`
    - don't need owner permissions to upload/download from the S3 bucket.
    - make sure you have region and access tokens listed in ~/.aws/ folder

* `6.0.0`
    - Updated to the latest versions of aws-sdk and maven-wagon.
    - Changed order of aws credential resolution strategy.
    - Added support of all regions defined in aws-sdk.

## License

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
