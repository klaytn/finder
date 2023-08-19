# KlaytnFinder

KlaytnFinder is a website that provides information about blocks, transactions, tokens, NFTs, contracts, and more on the Klaytn blockchain.

## 1. Structure

### 1.1. Project Structure

It consists of five repositories.

1. [finder-infra](https://github.com/klaytn/finder-infra.git): Infrastructure-related code. Configured using Terraform and deployed on AWS.
2. [finder-spark](https://github.com/klaytn/finder-spark.git): Data processing-related code. Written in Scala and Spark. Refines data received from EN Nodes and stores it in the database.
3. [finder-api](https://github.com/klaytn/finder-api.git): Backend-related code. Written in Kotlin and utilizes the Spring Boot framework. Provides data from the database via APIs. Comprises modules: [module-common](https://github.com/klaytn/finder-api/tree/main/module-common), [module-compiler-api](https://github.com/klaytn/finder-api/tree/main/module-compiler-api), [module-domain](https://github.com/klaytn/finder-api/tree/main/module-domain), [module-front-api](https://github.com/klaytn/finder-api/tree/main/module-front-api), [module-gateway\[DEPRECATED\]](https://github.com/klaytn/finder-api/tree/main/module-gateway), [module-open-api](https://github.com/klaytn/finder-api/tree/main/module-open-api), [module-worker](https://github.com/klaytn/finder-api/tree/main/module-worker).
4. [finder-helm-chart](https://github.com/klaytn/finder-helm-chart.git): Kubernetes-related code. Configured using Helm.
5. [finder-fe](https://github.com/klaytn/finder-fe.git): Frontend-related code. Written in React.

### 1.2. Architecture

![KlaytnFinder Architecture](https://cdn.klaytnfinder.io/klaytnfinder-structure.png)

### 1.3. Technology Stack

1. **Infrastructure**: AWS, Terraform
2. **Data Processing**: Scala, Spark, SQL, Hikari
3. **Backend**: Kotlin, Spring Boot, JPA, Gradle
4. **Kubernetes**: Helm, Kustomize
5. **Frontend**: React, TypeScript, Styled Components, Turbo

### 1.4. Resources

1. **EN Node (Full Archive)**: EN Node provides information about blocks, transactions, tokens, NFTs, and contracts on the Klaytn blockchain. It sends topics to Kafka through [chaindatafetcher](https://github.com/klaytn/klaytn/tree/dev/datasync/chaindatafetcher).
2. **Kafka**: A data streaming platform that streams data from chaindatafetcher to Spark.
   Receives four topics as described below (may vary based on [EN Node settings](https://github.com/klaytn/klaytn/blob/d8c3b98ef3f899d6d941c03b717b8be26e20127f/cmd/utils/flags.go), [topic name creation rules](https://github.com/klaytn/klaytn/blob/e743a2c1b81031da95bee3a7a06a658ae2686e07/datasync/chaindatafetcher/kafka/config.go#L111)).

   - `cypress.klaytn.chaindatafetcher.en-0.blockgroup.v1`
   - `baobab.klaytn.chaindatafetcher.en-0.blockgroup.v1`
   - `cypress.klaytn.chaindatafetcher.en-0.tracegroup.v1`
   - `baobab.klaytn.chaindatafetcher.en-0.tracegroup.v1`

3. **Spark**: A data processing engine that refines data from Kafka and stores it in the database. Can handle batch or streaming processing.
4. **MySQL**: Stores data processed by Spark. Consists of one instance for Baobab and seven instances for Cypress.
5. **Redis**: Backend cache server that stores Spark processing status. Sends processed data from Spark to the backend using Pub/Sub.
6. **Elasticsearch**: Backend search engine. Facilitates searching for Account, Token, NFT, Contract, Transaction data.
7. **S3**: Comprises five buckets.

   - **`klaytn-prod-lake`**: Stores data received from Kafka.
     - `/klaytn/{cypress | baobab}`
       - `label=kafka_log/`
         - `topic=block/`: Stores block data. Partitioned in 100,000-block segments.
         - `topic=trace/`: Stores Internal Transaction data. Partitioned in 100,000-block segments.
   - **`klaytn-prod-spark`**: Stores data processed by Spark.

     - `emr/`: Stores EMR-related settings and logs.
       - `{Step_ID}/`: EMR Step ID
     - `jars/`: Stores .jar files for Spark applications.
       - `{User_ID}`: .jar files uploaded by users, organized by folders named in the _YYYYMMDD_HH_ format.
     - `jobs/`: Created during Spark Application Streaming Job processing.
       - `{Class_Name}`: Represents the executed Streaming Class.
       - `if-want-stop-delete-this-file.prod-{cypress | baobab}.txt`: Delete this file to stop the Streaming Job.
       - `kafka-offset.prod-{cypress | baobab}.txt`: Stores Kafka Offset used by the Streaming Job.
     - `output/`: Stores batch-processed data from Spark. Formatted for direct loading from the database.
       - `{prod | dev | stag}-{cypress | baobab}/`: Differentiates by environment and network.
         - `AccountBatch`: Stores Account data (SCA, EOA). Created during `AccountMakeData` batch execution, partitioned in 100,000-block segments.
         - `loadDataFromS3`: Created during `BlockMakeData` and `InternalTXMakeData` batch execution, partitioned in 100,000-block segments.
           - `block/`: Stores block data in `blocks` table format.
           - `eventlog/`: Stores Event Log data in `event_logs` table format.
           - `list/`: Stores file locations to be loaded and table names.
             - `block/`
             - `trace/`
           - `trace_index_merge/`: Combines smaller Account-specific Internal Transaction data into a format that can be loaded from the database.
           - `trace_index/`: Stores Account-specific Internal Transaction data in `internal_transaction_index` table format.
           - `trace_merge/`: Combines smaller Internal Transaction data into a format that can be loaded from the database.
           - `trace/`: Stores Internal Transaction data in `internal_transactions` table format.
           - `tx/`: Transaction data is stored in the form of the `transactions` table.

   - **`klaytn-prod-finder-private`**: Stores files used by the backend.
     - `compiler/`: Stores compilers for different versions of Solidity (used by compiler-api).
       - `linux-amd64/`: Linux version (Server-Pod)
       - `macosx-amd64/`: macOS version (Local)
     - `finder/`: Stores files used by the Worker API.
       - `{cypress | baobab}/`
         - `proposed-blocks/`
         - `nft-inventory-contents/`
     - `maintenance/`: Stores files used by the Front API.
       - `{cypress | baobab}-status.json`: Contains content to be displayed during server maintenance.
       - `status.json`
   - **`klaytn-prod-finder-public`**: Stores files used by the frontend.
     - web: Stores React Build results for Production environment (connected via Cloud Front).
       - `{cypress | baobab}/`
     - web-stage: Stores React Build results for Production environment (used by EKS Pods).
       - `{cypress | baobab}/`
   - **`klaytn-prod-finder-common`**: Stores files commonly used (served as CDN via Cloud Front).
   - **`klaytn-prod-apnortheast2-tfstate`**: Stores Terraform state.

---

## 2. Environment Setup and Operations

### 2.1. Environment Setup

The following should be installed by default.

- kubectl (`brew install kubectl`)
- kubectx (optional) (`brew install kubectx`)
- awscli (`brew install awscli`)
- session-manager-plugin (`brew install --cask session-manager-plugin`)
- cask (optional) (`brew install cask`)
- aws-vault (`brew install aws-vault`)
- docker
- docker-compose
- lens
- terraform (`brew install terraform`)
- zkCli (`brew install zookeeper`)

#### 2.1.1. Infrastructure (AWS)

- Set up your AWS account.
- Initialize aws-vault.

#### 2.1.2. Data Streaming (Scala, Spark)

- Install Java (`brew install --cask adoptopenjdk11`)
- Install Scala (`brew install scala`)
- Install Apache Spark (`brew install apache-spark`)
- Install sbt (`brew install sbt`)
- Install jq (`brew install jq`)
- Install IntelliJ IDEA (optional)

#### 2.1.3. Backend (Kotlin, Spring Boot)

- Install OpenJDK (`brew install openjdk`)
- Install Kotlin (`brew install kotlin`)
- Install Gradle (`brew install gradle`)

#### 2.1.4. Kubernetes (Helm)

- Setup kubeconfig (`aws eks update-kubeconfig --region region-code --name my-cluster`)
- Setup kubecontext (`kubectx my-cluster`)

#### 2.1.5. Frontend (React)

- Install node canvas (`brew install pkg-config cairo pango libpng jpeg giflib librsvg`)
- Install Node.js (`brew install node`)
- Install and setup nvm
- Set the nvm version (`nvm use`) -> If not installed yet, install the version (`nvm install {version}`)

---

## 2.2. Operations

#### 2.2.1. Infrastructure (AWS)

[Readme Link](https://github.com/klaytn/finder-infra/blob/main/README.md)

#### 2.2.2. Data Streaming (Scala, Spark)

[Readme Link](https://github.com/klaytn/finder-spark/blob/main/README.md)

#### 2.2.3. Backend (Kotlin, Spring Boot)

[Readme Link](https://github.com/klaytn/finder-api/blob/main/README.md)

#### 2.1.4. Kubernetes (Helm)

[Readme Link](https://github.com/klaytn/finder-helm-chart/blob/main/README.md)

#### 2.1.5. Frontend (React)

[Readme Link](https://github.com/klaytn/finder-fe/blob/main/README.md)
