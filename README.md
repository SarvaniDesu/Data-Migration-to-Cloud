# Real-time Migration of Hotel System to Cloud Infrastructure
This project employs an advanced multicloud approach, utilizing the capabilities of both GCP and AWS, to provide a sophisticated Infrastructure as Code (IAC) solution seamlessly integrated with Terraform. In a practical scenario, a luxury hotel is undergoing the migration of its COVID guest testing application and infrastructure to the cloud.

## Requirements

- **AWS and GCP Accounts**
  
  - Create a user "terraform-pt-1" in the AWS IAM service, with programmatic access (CLI) and the "AmazonS3FullAccess" policy.
  - Create an access key for it, download, rename the CSV to "accessKeys.csv," and place it in the "mission1" directory.
  
  - Create a user "luxxy-covid-testing-system-pt-app1" in the AWS IAM service, with programmatic access (CLI) and the "AmazonS3FullAccess" policy.
  - Create an access key for it, download, rename the CSV to "luxxy-covid-testing-system-pt-app1.csv," and place it in the "mission2" directory.
  
  - Change the bucket name in the tcb_aws_storage.tf file (mission1 folder) to your desired S3 bucket.
 
## Stack

- **Infrastructure as Code (IAC):** Terraform
- **Storage:** AWS S3
- **Database:** Google Cloud SQL
- **Application:** Hosted and processed on Google Kubernetes Engine (GKE) using Google Container Registry (GCR)
- **Tools:**
  - Google Cloud Shell
  - Google Cloud Editor
 
## Step 1: Providing the Infrastructure

Run the following commands on Google Cloud PowerShell:

### Preparing the Files

```bash
mkdir mission1_pt
mv mission1.zip mission1_pt  # (if you uploaded the unzipped folder, just remove .zip in this command)
cd mission1_pt
unzip mission1.zip  # (if you uploaded the unzipped folder, ignore this step)
mv ~/accessKeys.csv mission1/pt
cd mission1/pt
chmod +x *.sh
```

### Preparing the environment

./aws_set_credentials.sh accessKeys.csv
gcloud config set project [your-project-id]

Replace [your-project-id] with your actual Google Cloud project ID.

### Setting the project on GCP

./gcp_set_project.sh

### Enabling Container Registry, CloudSQL, Kubernetes

gcloud services enable containerregistry.googleapis.com
gcloud services enable container.googleapis.com
gcloud services enable sqladmin.googleapis.com

### Providing the infrastructure

cd ~/mission1_pt/mission1/pt/terraform/
terraform init
terraform plan
terraform apply
During the terraform apply phase, you may be prompted to type "yes" to confirm the deployment. Please proceed by entering "yes" when prompted.

## Step 2: Preparing the SQL Network

Follow these steps to configure the network settings for your Cloud SQL instance:

1. Select your Cloud SQL instance.

2. Navigate to the "Connections" section.

3. Choose the "Networking" tab and, in "Instance IP assignment," enable Private IP.

4. Under "Associated Network," select “default.”

5. Click on "Set up connection."

6. If prompted, enable the Service Networking API.

7. Select the option "Use an automatically allocated IP range in your network."

8. Click "Continue" and then "Create connection."

9. Go to "Authorized Networks" and click "Add Network."

10. In "New Network," enter the following information:
    - Name: Public Access (For Testing Only)
    - Network: 0.0.0.0/0

11. Click "Done."

## Step 3: Converting the Application to Run in a Multicloud Environment

1. **Create a database user and password** in the Google Cloud SQL service.

2. **Connect to Google Cloud Shell** and execute the following commands:

    ```bash
    cd ~
    mkdir mission2_en
    cd mission2_en
    wget https://tcb-public-events.s3.amazonaws.com/icp/mission2.zip
    unzip mission2.zip
    ```

3. **Connect to the SQL instance** with the command:

    ```bash
    mysql --host=<your_public_ip_cloudsql> --port=3306 -u app -p
    ```

    Replace `<your_public_ip_cloudsql>` with your actual Cloud SQL instance's public IP.

4. **Create the "records" table** with the following commands:

    ```sql
    use dbcovidtesting;
    source ~/mission2_en/mission2/en/db/create_table.sql
    ```

5. **Validate the table creation:**

    ```sql
    show tables;
    exit;
    ```

6. **Enable Google Cloud Build** with the command:

    ```bash
    gcloud services enable cloudbuild.googleapis.com
    ```

## Step 4: Build the Docker Image and Edit the Kubernetes YAML File

1. **Build the Docker image:**

   Execute the following commands:

    ```bash
    cd ~/mission2_en/mission2/en/app
    gcloud builds submit --tag gcr.io/<PROJECT_ID>/luxxy-covid-testing-system-app-en
    ```

   Replace `<PROJECT_ID>` with your actual Google Cloud project ID.

2. **Modify the `luxxy-covid-testing-system.yaml` file:**

   Edit the file in the following sections:

   - `image`: Insert your project ID.

   - `AWS_BUCKET`: Insert the name of your already created bucket.

   - `S3_ACCESS_KEY`: Insert according to the `luxxy-covid-testing-system-en-app1.csv` file.

   - `S3_SECRET_ACCESS_KEY`: Insert according to the `luxxy-covid-testing-system-en-app1.csv` file.

   - `DB_HOST_NAME`: Insert the private IP of your database instance.

## Step 5: Deploy the Application on the Kubernetes Cluster

1. Access GKE through the console, select the "Connect > Run in Cloud Shell" option, and execute the command to authenticate Kubernetes.

2. Perform the deployment with the following command:

    ```bash
    cd ~/mission2_en/mission2/en/kubernetes
    kubectl apply -f luxxy-covid-testing-system.yaml
    ```

3. In the GKE panel, go to "Services and Ingress" and find the IP of the generated endpoint.

4. The application is up and running!

## Step 6: Migration of Application Data and Files

1. **In Google Cloud Shell, execute the following commands:**

    ```bash
    cd ~
    mkdir mission3_pt
    cd mission3_pt
    wget https://tcb-public-events.s3.amazonaws.com/icp/mission3.zip
    unzip mission3.zip
    ```

    **Connect to the database instance:**

    ```bash
    mysql --host=<public_ip_address> --port=3306 -u app -p
    ```

    **Import the exported data from the on-premises database:**

    ```sql
    use dbcovidtesting;
    source ~/mission3_pt/mission3/pt/db/db_dump.sql
    ```

2. **In AWS CloudShell, download the files using the following commands:**

    ```bash
    mkdir mission3_pt
    cd mission3_pt
    wget https://tcb-public-events.s3.amazonaws.com/icp/mission3.zip
    unzip mission3.zip
    ```

    **Synchronize the downloaded data with S3:**

    ```bash
    cd mission3/pt/pdf_files
    aws s3 sync . s3://luxxy-covid-testing-system-pdf-pt-<your_bucket>
    ```

3. **Test the application in the "View Records" tab** to verify if the data and files are available.
