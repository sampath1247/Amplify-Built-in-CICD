# Demo for Amplify Built-in CICD

<div style="color: red; font-size: 20px; font-weight: bold; border: 2px solid red; padding: 10px; background-color: #ffe6e6;">
⚠️ WARNING: ***** DO NOT TRY THIS DEMO *****
Ensure that sensitive information like OAuth tokens is never hardcoded in your source code. Always use secure storage solutions like AWS Secrets Manager. 
</div>

When dealing with OAuth tokens, especially when integrating with third-party services such as **GitHub**, **best practices** require you to securely handle and store sensitive information like the **OAuth token**. In the context of creating an AWS Amplify App via **AWS CDK**, the **OAuth token** should not be hard-coded in your source code, as this can lead to security risks such as exposing secrets.

Instead, the recommended approach is to store the OAuth token securely and reference it from a secure location like **AWS Secrets Manager**. This ensures that the token is not exposed in the code and is only accessible by authorized services.

### Best Practice for Handling GitHub OAuth Token with AWS CDK

1. **Store the GitHub OAuth Token in AWS Secrets Manager**: 
   Use **AWS Secrets Manager** to securely store your GitHub OAuth token. Secrets Manager encrypts the token and allows controlled access via IAM roles and policies.

2. **Access the Token from AWS CDK**: 
   In your CDK code, use the AWS SDK to retrieve the OAuth token from **Secrets Manager** and pass it to the **Amplify App** resource.

---

### Step-by-Step Approach

#### 1. **Store GitHub OAuth Token in AWS Secrets Manager**

Follow the [github mannual](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens) to generate a classic token for repo access. Then:

- Navigate to the **AWS Secrets Manager** console.
- Click **Store a new secret**.
- Choose **Other type of secret**. In the key/value pair, assign the key as `github-oauth-token`, assign the value as your Github token.
- Assign a **secret name** (e.g., `GitHubOAuthToken`).
- Save the secret.

You can also do this programmatically using the **AWS SDK** or AWS CDK by creating a secret in Secrets Manager.

Please **note**: AWS Secrets Manager is only in free trial - it's free for 30 days since you store your first secret.

Here is a code sample to retrieve the secrete in your application by using SDK:
```java
// Use this code snippet in your app.
// If you need more information about configurations or implementing the sample
// code, visit the AWS docs:
// https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/home.html

// Make sure to import the following packages in your code
// import software.amazon.awssdk.regions.Region;
// import software.amazon.awssdk.services.secretsmanager.SecretsManagerClient;
// import software.amazon.awssdk.services.secretsmanager.model.GetSecretValueRequest;
// import software.amazon.awssdk.services.secretsmanager.model.GetSecretValueResponse;	

public static void getSecret() {

    String secretName = "GitHubOAuthToken";
    Region region = Region.of("us-east-1");

    // Create a Secrets Manager client
    SecretsManagerClient client = SecretsManagerClient.builder()
            .region(region)
            .build();

    GetSecretValueRequest getSecretValueRequest = GetSecretValueRequest.builder()
            .secretId(secretName)
            .build();

    GetSecretValueResponse getSecretValueResponse;

    try {
        getSecretValueResponse = client.getSecretValue(getSecretValueRequest);
    } catch (Exception e) {
        // For a list of exceptions thrown, see
        // https://docs.aws.amazon.com/secretsmanager/latest/apireference/API_GetSecretValue.html
        throw e;
    }

    String secret = getSecretValueResponse.secretString();

    // Your code goes here.
}
```

#### 2. **Update CDK Code to Access the Token from Secrets Manager**

##### 2.1. **Prerequisites**:
Before we begin, make sure you have the following:
- **AWS CLI** installed and configured on your machine.
- **Java Development Kit (JDK)** installed on your machine (for AWS SDK and CDK).
- **VS Code** installed.
- **Node.js and npm** installed (for AWS Amplify setup).
- **AWS CDK** installed: `npm install -g aws-cdk`.

You’ll need an **AWS account** and the required permissions to deploy AWS resources like Amplify, Lambda, API Gateway, etc.

---

##### 2.2. **Setting up the Project in VS Code**:
In this demo, we will use AWS CDK for infrastructure and **AWS SDK for Java** to interact with the Lambda function. Let’s create a new CDK project in Java.

###### Step 1: Create a new CDK Project
0. Prerequisites
**Install Node.js**: Download and install Node.js from [nodejs.org](https://nodejs.org/).
If you haven't install AWS CLI:
- Download and install the AWS CLI from [AWS CLI Installation Guide](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html).
    - Configure it with your credentials:

```bash
aws configure
```
**Install AWS CDK**:

```bash
npm install -g aws-cdk
```
**Install Maven**: Download Maven from [Maven's website](https://maven.apache.org/) if not already installed.

1. **Set up VS Code**: Install the Java Extension Pack in VS Code. Open **VS Code**.

2. Open the terminal and run the following command to create a new CDK Java project:
   ```bash
   mkdir amplify-textract-demo
   cd amplify-textract-demo
   cdk init app --language java
   ```

3. Add required CDK dependencies to your `pom.xml`:
   ```xml
   <dependencies>
       <dependency>
           <groupId>software.amazon.awscdk</groupId>
           <artifactId>aws-cdk-lib</artifactId>
           <version>2.x.x</version> <!-- Replace with latest CDK version -->
       </dependency>
       <dependency>
           <groupId>software.amazon.awscdk</groupId>
           <artifactId>core</artifactId>
           <version>2.x.x</version> <!-- Replace with latest CDK version -->
       </dependency>
       <dependency>
           <groupId>software.amazon.awscdk</groupId>
           <artifactId>aws-amplify</artifactId>
           <version>2.x.x</version> <!-- Replace with the latest version of CDK AWS Amplify module -->
       </dependency>
           <!-- AWS Secrets Manager CDK library -->
       <dependency>
           <groupId>software.amazon.awscdk</groupId>
           <artifactId>aws-secretsmanager</artifactId>
           <version>2.0.0</version> <!-- Replace with the latest version -->
       </dependency>
   
       <!-- AWS IAM library to handle permissions (optional but recommended) -->
       <dependency>
           <groupId>software.amazon.awscdk</groupId>
           <artifactId>aws-iam</artifactId>
           <version>2.0.0</version> <!-- Replace with the latest version -->
       </dependency>
   </dependencies>
   ```

4. Run `mvn clean install` to install the dependencies.

###### Step 2: Create Your Lambda Function in the CDK

In your AWS CDK project, you can use **AWS Secrets Manager** to access the OAuth token and inject it into the Amplify App resource.

Here’s an updated version of the CDK code, which retrieves the GitHub OAuth token from Secrets Manager and passes it to the Amplify app:

#### Code: `AmplifyStack.java`

```java
package com.myorg;

import software.amazon.awscdk.core.*;
import software.amazon.awscdk.services.amplify.*;
import software.amazon.awscdk.services.secretsmanager.*;
import software.amazon.awscdk.services.iam.*;

public class AmplifyStack extends Stack {

    public AmplifyStack(final Construct scope, final String id) {
        super(scope, id);

        // 1. Retrieve the GitHub OAuth Token from Secrets Manager
        Secret githubToken = Secret.fromSecretNameV2(this, "GitHubOAuthToken", "github-oauth-token");

        // 2. Create an Amplify App connected to the GitHub repository
        CfnApp amplifyApp = CfnApp.Builder.create(this, "AmplifyApp")
            .name("MyAmplifyApp")
            .repository("https://github.com/your-username/your-repository-name")  // Replace with your GitHub repository URL
            .oauthToken(githubToken.secretValue().toString()) // Securely access the token from Secrets Manager
            .build();

        // 3. Create a branch for the Amplify App (e.g., main branch)
        CfnBranch.Builder.create(this, "AmplifyBranch")
            .appId(amplifyApp.getAttrAppId())
            .branchName("main")  // Branch in your GitHub repository
            .build();
    }
}
```

### Explanation of Key Steps:

1. **Retrieve OAuth Token from Secrets Manager**: 
   - The token is stored securely in **AWS Secrets Manager** with the name `github-oauth-token`. 
   - In the CDK code, the `Secret.fromSecretNameV2` method is used to securely reference the secret. This way, the token is never hardcoded in the source code.

2. **OAuth Token Injection**: 
   - The retrieved GitHub OAuth token is then passed to the `CfnApp` constructor in the `oauthToken` field.
   - This ensures that the OAuth token is securely retrieved from Secrets Manager and used only at runtime.

3. **Amplify App Creation**: 
   - The `CfnApp` resource is used to create an Amplify app and associate it with the GitHub repository.
   - The `CfnBranch` resource creates a branch for the app (e.g., `main` branch).

### 3. **Configure IAM Permissions**

Ensure that the **CDK execution role** (which is the IAM role used by CDK to deploy resources) has permission to access the secret stored in **Secrets Manager**.

You can add the following IAM policy in your CDK stack:

```java
// Create an IAM policy to allow reading from Secrets Manager
PolicyStatement secretPolicy = PolicyStatement.Builder.create()
    .actions(List.of("secretsmanager:GetSecretValue"))
    .resources(List.of(githubToken.getSecretArn()))
    .build();

// Attach the policy to the CDK role
Role cdkExecutionRole = Role.fromRoleArn(this, "CdkExecutionRole", "arn:aws:iam::account-id:role/CDKExecutionRole");
cdkExecutionRole.addToPolicy(secretPolicy);
```

This ensures that your CDK execution role has the appropriate permissions to retrieve the GitHub OAuth token stored in Secrets Manager.

### 4. **Deploy the CDK Stack**

To deploy the updated stack with the GitHub OAuth token stored securely in Secrets Manager:

1. **Bootstrap the environment** (if it’s your first deployment):

```bash
cdk bootstrap
```

2. **Deploy the CDK stack**:

```bash
cdk deploy
```

This will:
- Create the **Amplify App**.
- Retrieve the OAuth token from **Secrets Manager**.
- Create the **Amplify branch** and configure the connection to your GitHub repository.

---

