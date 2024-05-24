# Infrastructure Project Writeup: Credit Engine API for Private Lending Fintech Company
## Introduction
### Project Overview
The Credit Engine API was developed to automate the retrieval of credit data for a private lending fintech company specializing in higher education finance. Automating this process enhances the speed and accuracy of loan underwriting.
### Objectives
- Automate credit pulls to streamline loan application processing
- Reduce human error and increase data security during credit report handling
- Facilitate the secure transfer of credit report data between third-party services
## Domain and Product Description
### Domain Overview
The private lender for which the Credit Engine API was built operates within the financial technology (fintech) domain; specifically, the educational finance sector. This setor emphasizes the use of advanced technology and data modelling to offer innovative, affordable financial products pertaining to the pursuit of education. 
### Product Overview
The Credit Engine API automates the extraction and processing of credit report data directly from a major credit bureau when a student applies for a loan with this private lender. This automation drastically speeds up the credit check step of the underwriting process by eliminating manual credit pulls, data entry, and risk calculation. This increased operational efficiency streamlines loan application processing and decision-making, providing a superior user experience for both loan officers and applicants.
### Target Audience
Primary users include loan underwriters within the company, leveraging the API to improve efficiency and accuracy in loan approvals. Secondary beneficiaries are college students applying for loans, who experience faster processing and enhanced security of their personal information.
## Problem Statement
### Challenges and Requirements
The API addresses significant challenges, including the effort involved in manually pulling credit reports from a bureau's online portal, the associated risks such as data entry errors and security vulnerabilities, as well as the manual review and evaluation of credit data.
### Business and Technical Requirements
#### Business Requirements: 
Cost-effective operations, scalability to handle growth, compliance with data protection laws, timely market delivery, and intuitive user experience.
#### Technical Requirements: 
Secure data transfer mechanisms, seamless integration with existing systems, robust data management and integrity safeguards, and specific technology stack choices.
## System Architecture & Design 
The project's architecture is designed to automate credit pulls efficiently and securely by integrating Salesforce, the Credit Engine API, and the TransUnion API. 
<!-- This streamlined process enhances data accuracy and speeds up the loan approval process within our fintech framework. -->
### Process Flow
1. **Salesforce Activation** 
    > Moving a Salesforce Opportunity's* Stage into "Pull Credit" triggers a webhook which passes applicant data from Salesforce to the configured API endpoint (the CreditEnginePublisher API).
2. **Publisher Checks for Previous Credit Pulls**
    > The Credit Engine Publisher queries an AWS DynamoDB table responsible for recording credit pull activity, checking for previous pulls on the present applicant id.
    > * If id is not found in DB a new table record is created, containing the applicant id and date, time of credit pull
    > * If id is found in DB, the process is terminated and an error is thrown
3. **Publisher Sends Applicant Data to AWS SQS**
    > For new credit pulls, the Publisher proceeds to pass the applicant data recieved from Salesforce to an AWS SQS Queue (message broadcaster).
    > * Messages (applicant data) are queued in the order recieved
    > * A new message added to the queue triggers an invocation of the Credit Engine API
4. **Credit Engine API Initiates Request** 
    > Credit Engine packages applicant data into XML request body and securely sends request to TransUnion API using PEM certificate validation within a whitelisted AWS VPC subnet.
5. **TransUnion Interaction** 
    > TransUnion API returns requested credit report data.
6. **Credit Engine API Processing and Upload**
> *  Credit Engine parses response data, applies business logic for evaluating applicant’s loan eligibility
> *  Uploads processed credit report data to originating Salesforce Opportunity

\* An 'Opportunity' in Salesforce refers to an applicant’s underwriting record

### Sequence Diagrams
The system sequence diagram below depicts the flow of data between system components:
![Credit Engine API System Sequence Diagram](./docs/system_sequence_diagram.png)

The API sequence diagram below depicts the data flow within the Credit Engine API as well as its interactions with third-party integrations:
![Credit Engine API Sequence Diagram](./docs/api_sequence_diagram.png)


## Technology Stack
### AWS Services
- ECR for storing Docker container images of the API
- Lambda for serverless, event-driven execution
- DynamoDB for high-performance NoSQL data management 
- API Gateway for secure publishing of API
- SQS for message queueing
- IAM for resource access control
- VPC for network isolation
### Development Tools
- Git for version control
- Docker for containerization
- CircleCI for CI/CD
- Terraform Cloud for infrastructure management
### Third-Party Platforms
- TransUnion API for retrieving applicants’ credit report data
- Salesforce CRM for retrieving applicant data and storing credit pull results

## Infrastructure & Security
### Deployment
The deployment of our Credit Engine API is designed to maximize automation and reliability using CircleCI, AWS Lambda, and AWS ECR. This setup facilitates a robust, automated pipeline that ensures code is thoroughly tested, securely containerized, and deployed seamlessly to a serverless environment.

#### CI/CD Pipeline:
- Automated Testing and Coverage: Code commits trigger automated workflow in CircleCI; tests and coverage reports are run to ensure quality and reliability
- Containerization: Upon successful testing, code is containerized using Docker; Docker image is pushed to AWS ECR (managed Docker container registry), ensuring secure and versioned storage of application images
- Lambda Deployment: Finally, AWS Lambda function is updated with latest Docker image from AWS ECR
### Networking
#### Lambda Configuration within VPC:
- VPC Integration: Credit Engine Lambda function operates within specific AWS VPC private subnet, allowing it to successfully execute requests to TransUnion API via static, whitelisted ip address
- Network Isolation and Security: Running within VPC subnet isolates Credit Engine from public internet access, controlling exposure and access through defined security groups
#### Event-Driven Invocation:
- SQS as Trigger: 
> - Credit Engine Lambda function is triggered by messages arriving in an AWS SQS queue
> - Each new message initiates separate instance of Lambda function, effectively handling incoming events in parallel and ensuring scalable, responsive processing of tasks as they arrive
- Scalable Event Handling: 
> - This setup facilitates automatic scaling of Lambda according to volume of incoming messages
> - AWS manages instantiation of function instances based on rate and number of messages in the SQS queue
#### Security and Compliance:
- AWS VPC and Subnets: Utilizes AWS VPC, subnets for network isolation, enhancing controlled access through whitelisting of IP ranges
- PEM Certificate Validation: Employs PEM certificate validation for data encryption, ensuring secure data transmission with external APIs
- IAM Roles and Policies: Applies IAM roles to restrict resource access within AWS to authorized services and personnel only, allowing secure interaction among services like Lambda, SQS, and ECR while adhering to the principle of least privilege
