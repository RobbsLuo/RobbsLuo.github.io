---
title: Installing and Using DynamoDB on Ubuntu
date: 2016-11-16 10:28:46
tags:
  - DynamoDB
  - Software
categories:
  -  Software
description: A record of installing DynamoDB on Ubuntu
lang: en
---

## Installation
1. #### Java SDK
   ```bash
    aptitude install openjdk-8-jdk
   ```

2. ####   Install the extraction tool
   ```bash
    # Install the extraction tool
    aptitude install unzip
   ```

3. #### Download and install DynamoDB
   ```bash
    # Download the archive
    wget http://dynamodb-local.s3-website-us-west-2.amazonaws.com/dynamodb_local_latest.zip

    # Extract
    unzip ./dynamodb_local_latest.zip

    # Start
    java -Djava.library.path=./DynamoDBLocal_lib -jar DynamoDBLocal.jar -sharedDb

    # View help
    java -Djava.library.path=./DynamoDBLocal_lib -jar DynamoDBLocal.jar -help
   ```

4. #### javascript shell
   [http://127.0.0.1:8000/shell/](http://127.0.0.1:8000/shell/)
