maven-s3-repo-uploader
================

## Problem to solve
* You chose S3 as the private maven repository.
* You can deploy and download from S3 repository via extensions such as [s3-wagon-private](https://github.com/technomancy/s3-wagon-private) or [maven-s3-wagon](https://github.com/jcaddel/maven-s3-wagon) in Maven or Leiningen project.
* However, Leiningen doesn't have a function to upload an artifact from a local file system. [Maven provides the way to deploy third party JARs](https://maven.apache.org/guides/mini/guide-3rd-party-jars-remote.html), but non-standard protocol such as `s3` doesn't work out-of-the-box. The document suggests to put wagon-provider to `$M2_HOME/lib`, but it requires manual configuration.

# Pre-requisites
* Maven. Tested with `3.3.9`
* AWS CLI. Tested with `aws-cli/1.10.19`

## Set up
* Create a S3 bucket. 

```
aws s3 mb s3://my-private-repo
```
* Configure the access control against the S3 bucket. See [AWS document](http://docs.aws.amazon.com/AmazonS3/latest/dev/s3-access-control.html) for the details. 
* Obtain IAM user's access key and secret key that are used to access S3 bucket .
* Store the key information securely. First, create a master password. For the details, please refer to [Maven Password Encryption](https://maven.apache.org/guides/mini/guide-encryption.html).

```
$ mvn --encrypt-master-password
```

Save the output in `$HOME/.m2/settings-security.xml`. 

```
<settingsSecurity>
    <master>{generatedmasterpasswordhere}</master>
</settingsSecurity>
```

* Encrypt AWS secret key. 

```
$ mvn --encrypt-password
```

* Store the plain access key id and encrypted password to `$HOME/.m2/settings.xml`. 

```
<settings>
    <servers>
        <server>
            <id>private-third-party</id>
            <username>YOURAWSACCESSKEYID</username>
            <password>{generatedpassphrasehere}</password>
        </server>
    </servers>
</settings>
```

## Upload
* Run the script as following. 

```
Usage:
./upload.sh groupId artifactId version packaging filePath [repository_id] [s3_url]
```

Example: 

```
./upload.sh datomic datomic-pro 0.9.5359 jar ~/Downloads/datomic-pro-0.9.5359/datomic-pro-0.9.5359.jar private-third-party s3://my_repo_bucket/third-party/
```

Please note the repository id must match with `id` specified in `$HOME/.m2/settings.xml`. 

* You can omit `repository_id` and `s3_url` command line arguments by creating `upload.env` and add the following entries. 

```
export REPOSITORY_ID=private-third-party
export REPOSITORY_URL=s3://my-private-repo/third-party/
```

## Implementation
* Extension needs to be specified to handle non-standard protocol. Achieved this by adding a simple 
  `pom.xml`. Note that the attributes such as `groupId` or `artifactId` are ignored. 
* `s3-wagon-private` has some classpath issue, so used [maven-s3-wagon](https://github.com/jcaddel/maven-s3-wagon) extension.
