# Train Equipment Diagnostics Simulator (AWS Project)

**Project Goal:** Simulate uploading sensor logs from train equipment to AWS. Automatically detect issues (e.g., overheating, voltage drops) and generate basic reports.

---

## 🧠 Project Goal

- Upload a `.csv` sensor log to **Amazon S3**  
- Automatically trigger an **AWS Lambda** function  
- Parse the file and check for system issues  
- Print results to **CloudWatch Logs**

<p align="center">
  <img 
    src="https://github.com/user-attachments/assets/927bbfe2-8009-4f93-95d5-088f61bb74b1" 
    height="50%" 
    width="50%" 
    alt="AWS_Train_Diagram_Screenshot"
  />
</p>

---

## 🧰 Services Used (All Free Tier)

- **Amazon S3** – Store and manage uploaded logs  
- **AWS Lambda** – Execute Python diagnostic logic  
- **Amazon CloudWatch Logs** – Monitor and view output  
- **IAM** – Set secure role and permissions for Lambda

---

## 📁 Step 1: Prepare Your Sensor Log File

Create a file named `sensor_log_01.csv` with the following content:

<p align="center">
  <img 
    src="https://github.com/user-attachments/assets/0838b038-c6ad-40fd-8983-bfb93cf76dde" 
    height="40%" 
    width="40%" 
    alt="sensor_log_01-Image"
  />
</p>

### ✅ Safe Sensor Readings (Last 3 Logs)

- 🌡️ **temperature**: `78°C` — ✅ safe  
- 🛠️ **brake_pressure**: `72 PSI` — ✅ safe  
- 🔋 **battery_voltage**: `25.0 V` — ✅ safe


## 🪣 Step 2: Create an S3 Bucket

1. In the AWS Console, open **Amazon S3**.
2. Click **Create bucket**.
3. Name it: `train-diagnostics-logs`.
4. Leave all defaults selected.
5. Click **Create bucket**.

<p align="center">
  <img 
    src="https://github.com/user-attachments/assets/b36a528b-732d-43e4-a43f-70bf85ac54cd" 
    height="70%" 
    width="70%" 
    alt="S3_Bucket_AWS_Screenshot_1"
  />
</p>

---

## ⬆️ Step 3: Upload the CSV File to S3

### Via AWS Console:
1. Open the `train-diagnostics-logs` bucket.
2. Click **Upload**.
3. Select `sensor_log_01.csv`.
4. Click **Upload**.

<p align="center">
  <img 
    src="https://github.com/user-attachments/assets/e07bd19f-c5aa-424f-9b5d-fcb67fc79f42" 
    height="70%" 
    width="70%" 
    alt="S3_CSV_Upload_AWS_Screenshot_2"
  />
</p>

---

## 🔐 Step 4: Create an IAM Role for Lambda

1. Go to **IAM** → **Roles** → **Create Role**.
2. Select **AWS service** → **Lambda**.
3. Click **Next** and attach the following policies:
   - `AmazonS3ReadOnlyAccess`
   - `CloudWatchLogsFullAccess`
4. Click **Next**.
5. Name the role: `lambda-s3-diagnostics-role`.
6. Click **Create role**.

<p align="center">
  <img 
    src="https://github.com/user-attachments/assets/9171212c-f7d1-4764-8b45-5e13d99ff83e" 
    height="60%" 
    width="60%" 
    alt="IAM_Lambda_AWS_Screenshot_1"
  />
</p>

---

## 🧠 Step 5: Create the Lambda Function

1. Navigate to **Lambda** → **Create Function**.
2. Set:
   - **Function name**: `TrainLogAnalyzer`
   - **Runtime**: `Python 3.12`
   - **Execution role**: Choose existing role → `lambda-s3-diagnostics-role`
3. Click **Create function**.

### 💻 Lambda Code (`lambda_function.py`):

```python
import boto3
import csv

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']

    response = s3.get_object(Bucket=bucket, Key=key)
    lines = response['Body'].read().decode('utf-8').splitlines()
    reader = csv.DictReader(lines)

    issues = []
    for row in reader:
        sensor = row['sensor_type'].strip().lower()
        value = float(row['value'])
        timestamp = row['timestamp']

        if sensor == 'temperature' and value > 90:
            issues.append(f"🔥 {timestamp} - HIGH TEMPERATURE: {value}°C")
        elif sensor == 'battery_voltage' and value < 23:
            issues.append(f"⚡ {timestamp} - LOW BATTERY VOLTAGE: {value}V")
        elif sensor == 'brake_pressure' and value < 50:
            issues.append(f"🛑 {timestamp} - LOW BRAKE PRESSURE: {value} PSI")

    if issues:
        print("🚨 Issues Found:")
        for issue in issues:
            print(issue)
    else:
        print("✅ No issues detected.")

    return {
        'statusCode': 200,
        'body': f"{len(issues)} issue(s) found." if issues else "No issues found."
    }
```
4. Click **Deploy** to save the function.

<p align="center">
  <img 
    src="https://github.com/user-attachments/assets/84bf5bea-0186-4cc4-a2ca-dc81ae3495f5" 
    height="60%" 
    width="60%" 
    alt="Lambda_Deploy_AWS_Screenshot_1"
  />
</p>

---

## 🔄 Step 6: Add the S3 Trigger to Lambda

1. In your **TrainLogAnalyzer** Lambda function, go to the **Configuration** tab.
2. Click **Triggers** → **Add trigger**.
3. Fill in the following details:
   - **Trigger type**: S3
   - **Bucket**: `train-diagnostics-logs`
   - **Event type**: `PUT`
   - **Suffix**: `.csv`
4. Click **Add**.

This configuration ensures that every time a `.csv` file is uploaded to the bucket, your Lambda function is triggered.

<p align="center">
  <img 
    src="https://github.com/user-attachments/assets/037213d8-cadc-4297-bde5-1f2f3c9c0cb1" 
    height="60%" 
    width="60%" 
    alt="Lambda_Trigger_Config_AWS_Screenshot_1"
  />
</p>

---

## 🧪 Step 7: Test the Workflow

Trigger the Lambda function by re-uploading the CSV file:

### Option 1: Via AWS Console
1. Open the `train-diagnostics-logs` bucket.
2. Click **Upload**, select `sensor_log_01.csv`, and upload it again.

### Option 2: Via AWS CLI
Run the following command:

```bash
aws s3 cp sensor_log_01.csv s3://train-diagnostics-logs/
```
---

## 📊 Step 8: View Results in CloudWatch Logs

1. Go to **Amazon CloudWatch** → **Logs** → **Log groups**  
2. Look for the log group: **/aws/lambda/TrainLogAnalyzer**
3. Click into the latest **Log stream** to view your Lambda function output.


<p align="center">
  <img 
    src="https://github.com/user-attachments/assets/e84de54d-0650-4524-a3dd-18e6937e55bc" 
    height="80%" 
    width="80%" 
    alt="CloudWatch_Logs_AWS_Screenshot_1"
  />
</p>

---

### 📌 Project Summary: *Train Equipment Diagnostics Simulator*

Designed a serverless solution to simulate analysis of train sensor logs for **Amtrak’s fleet**, using **AWS Lambda**, **Amazon S3**, and **CloudWatch Logs**. Built with **Python** to detect anomalies in temperature, battery voltage, and brake pressure — mirroring essential diagnostic tasks of a Service Engineer.

**Skills Used**:  
`AWS Lambda` · `Amazon S3` · `CloudWatch Logs` · `IAM` · `Python` · `CSV Parsing`

