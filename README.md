# Data Pipelining and Real-Time Data Streaming with Apache Flink and Flink SQL

## Objective
This project demonstrates a comprehensive pipeline to handle streaming data by integrating Confluent schemas, Kafka topics, Apache Flink, and Elasticsearch. The primary goal is to:

- Generate and process data using Confluent schemas.
- Feed the processed data into multiple Kafka topics.
- Stream and analyze the data in real-time using Apache Flink and Flink SQL.
- Create a dashboard for visualization and derive insights using Kibana.

---

## Dataset Description

### Overview of Payroll Datasets
These datasets represent structured payroll information for employees, their work locations, and bonus details. Each dataset originates from Confluent Avro schemas and has been transformed into JSON format for better readability and usage.

---

### Dataset Representations and Schemas

#### 1. Employee Information
- **Description**: Contains details about employees, including their personal information and hourly pay rates.

**Schema:**
```json
{
  "employee_id": "integer",       // Unique identifier for the employee
  "first_name": "string",        // First name of the employee
  "last_name": "string",         // Last name of the employee
  "age": "integer",              // Age of the employee
  "ssn": "string",               // Social Security Number of the employee
  "hourly_rate": "integer",      // Hourly rate in USD
  "gender": "string",            // Gender of the employee
  "email": "string"              // Email address of the employee
}
```
**Example Data:**
```json
{
  "employee_id": 1004,
  "first_name": "Hayley",
  "last_name": "Beatson",
  "age": 49,
  "ssn": "291-87-0461",
  "hourly_rate": 16,
  "gender": "female",
  "email": "ireg@mycompany.com"
}
```

#### 2. Employee Location
- **Description**: Tracks employee assignments to specific labs and departments, along with their arrival dates.

**Schema:**
```json
{
  "employee_id": "integer",       // Unique identifier for the employee
  "lab": "string",                // Lab assignment of the employee
  "department_id": "integer",     // Department ID where the employee is assigned
  "arrival_date": "integer"       // Date of arrival in lab (in days since epoch)
}
```
**Example Data:**
```json
{
  "employee_id": 1092,
  "lab": "lab-3",
  "department_id": 3,
  "arrival_date": 18053
}
```

#### 3. Employee Bonuses
- **Description**: Provides information about bonuses awarded to employees, including the bonus amount and timestamp of the transaction.

**Schema:**
```json
{
  "employee_id": "integer",       // Unique identifier for the employee
  "bonus": "integer",             // Bonus amount in USD
  "ts": "integer"                 // Timestamp of the bonus transaction (in milliseconds since epoch)
}
```
**Example Data:**
```json
{
  "employee_id": 1060,
  "bonus": 29,
  "ts": 1621784105514
}
```

---

## Data Pipeline Process

### 1. Data Generation
Data was generated from Confluent Avro schemas using the following commands:

```bash
./gendata.sh payroll_bonus.avro xyz1.json 10000
./gendata.sh payroll_employee.avro xyz2.json 10000
./gendata.sh payroll_employee_location.avro xyz3.json 10000
```
- **gendata.sh**: Extracts data from the Avro schema and converts it to JSON.
- **Input**: Avro schema.
- **Output**: JSON file with random synthetic data.

### 2. Data Transformation
The JSON files were transformed into key-value pairs using the `convert.py` script to prepare them for Kafka ingestion:

```bash
python $HOME/Documents/fake/convert.py
```

### 3. Kafka Ingestion
Data from the JSON files was streamed into Kafka topics using `gen_sample.sh`:

```bash
./gen_sample.sh /home/ashok/Documents/gendata/rev_xyz1.json | kafkacat -b localhost:9092 -t emp1 -K: -P
./gen_sample.sh /home/ashok/Documents/gendata/rev_xyz2.json | kafkacat -b localhost:9092 -t emp2 -K: -P
./gen_sample.sh /home/ashok/Documents/gendata/rev_xyz3.json | kafkacat -b localhost:9092 -t emp3 -K: -P
```
- **gen_sample.sh**: Streams transformed JSON data to Kafka topics.
- **Kafka Topics Created**: `emp1`, `emp2`, `emp3`.

### 4. Real-Time Analysis with Apache Flink
Apache Flink and Flink SQL were used for real-time streaming analysis.

---

#### Table Creation in Flink SQL

**1. Payroll Employee:**
```sql
CREATE TABLE payroll_employee (
    employee_id INT,
    first_name STRING,
    last_name STRING,
    age INT,
    ssn STRING,
    hourly_rate INT,
    gender STRING,
    email STRING
) WITH (
    'connector' = 'kafka',
    'topic' = 'emp1',
    'scan.startup.mode' = 'earliest-offset',
    'properties.bootstrap.servers' = 'kafka:9094',
    'format' = 'json',
    'json.timestamp-format.standard' = 'ISO-8601'
);
```

**2. Payroll Employee Location:**
```sql
CREATE TABLE payroll_employee_location (
    employee_id INT,
    lab STRING,
    department_id INT,
    arrival_date INT
) WITH (
    'connector' = 'kafka',
    'topic' = 'emp2',
    'scan.startup.mode' = 'earliest-offset',
    'properties.bootstrap.servers' = 'kafka:9094',
    'format' = 'json',
    'json.timestamp-format.standard' = 'ISO-8601'
);
```

**3. Payroll Bonus:**
```sql
CREATE TABLE payroll_bonus (
    employee_id INT,
    bonus INT,
    ts INT
) WITH (
    'connector' = 'kafka',
    'topic' = 'emp3',
    'scan.startup.mode' = 'earliest-offset',
    'properties.bootstrap.servers' = 'kafka:9094',
    'format' = 'json',
    'json.timestamp-format.standard' = 'ISO-8601'
);
```

#### 4. Enriched Views
A consolidated view was created by joining the tables:
```sql
CREATE VIEW enriched_payroll_employee_location_bonus AS
SELECT 
    PE.employee_id,
    PE.first_name,
    PE.last_name,
    PE.age,
    PE.ssn,
    PE.hourly_rate,
    PE.gender,
    PE.email,
    PEL.lab,
    PEL.department_id,
    PEL.arrival_date,
    PB.bonus,
    PB.ts
FROM payroll_employee AS PE
LEFT JOIN payroll_employee_location AS PEL ON PE.employee_id = PEL.employee_id
LEFT JOIN payroll_bonus AS PB ON PE.employee_id = PB.employee_id;
```

### 5. Data Insertion into Elasticsearch
Before creating the Kibana dashboard, the enriched data was inserted into an Elasticsearch index.

#### 1. Elasticsearch Table Creation
```sql
CREATE TABLE payroll_employee_dashboard (
    employee_id INT,
    first_name STRING,
    last_name STRING,
    age INT,
    ssn STRING,
    hourly_rate INT,
    gender STRING,
    email STRING,
    lab STRING,
    department_id INT,
    arrival_date INT,
    bonus INT,
    ts INT
) WITH (
    'connector' = 'elasticsearch-7',
    'hosts' = 'http://elasticsearch:9200',
    'index' = 'payroll_employee_dashboard',
    'format' = 'json'
);
```

#### 2. Data Insertion into Elasticsearch Index
```sql
INSERT INTO payroll_employee_dashboard
SELECT 
    employee_id,
    first_name,
    last_name,
    age,
    ssn,
    hourly_rate,
    gender,
    email,
    lab,
    department_id,
    arrival_date,
    bonus,
    ts
FROM enriched_payroll_employee_location_bonus;
```
## 6. Dashboard Creation with Kibana

- **Index Pattern:** Created an index pattern for `payroll_employee_dashboard` in Kibana.
- **Dashboard Design:** Visualized:
  - Employee details
  - Employee location details
  - Employee bonus distributions

---

# Payroll Employee Dashboard Analysis

This project visualizes and analyzes data from the `payroll_employee_dashboard` index using Kibana. Below is a detailed breakdown of the dashboard, key observations, and insights derived from the data.

### Snapshots

![dashboard img 1](https://github.com/user-attachments/assets/84558603-e73a-4fd4-adf0-5b9a0249b00e)
![dashboard img 2](https://github.com/user-attachments/assets/b7ba9bbc-1d28-49b1-baca-96ef673ec862)
![dashboard img 3](https://github.com/user-attachments/assets/ae720050-6997-43fe-8633-788fa498ab27)

---

### Video

https://github.com/user-attachments/assets/e92f9702-93b5-4065-a258-e7def84195d7

---

### 1. Age vs. Hourly Rate Analysis (Bar Chart)
- **Observation:** Employees aged 38 exhibit the most variability in hourly rates, with 7 unique rates.
- **Insights:** 
  - Employees aged 38 likely hold a diverse range of roles with varied pay structures.
  - Late 30s employees may occupy transitional or key positions.

### 2. Bonus Contribution by Gender (Bar Chart)
- **Observation:** Female employees contributed approximately 700,000 in bonuses, slightly more than male employees.
- **Insights:** 
  - Female employees may be more prominent in high-bonus-earning roles or departments.
  - Gender-based bonus disparity suggests potential role specialization or representation imbalance.

### 3. Top 5 Labs with the Highest Hourly Rates (Bar Chart)
- **Observation:** Labs 9, 2, 5, 6, and 7 have the highest average hourly rates, close to 16.
- **Insights:**
  - These labs likely house specialized or high-tier roles with competitive pay structures.
  - Consistency in hourly rates suggests standardized pay for critical operations.

### 4. Bonus Distribution by Age (Line Chart)
- **Observation:** Employees aged 31 and 49 received the highest cumulative bonuses.
- **Insights:**
  - Employees aged 31 may represent entry-level achievers, while those aged 49 reflect seasoned performers.
  - Bonuses reward early career promise and late-career excellence.

### 5. Correlation Between Hourly Rate and Bonus (Bar Chart)
- **Observation:** Bonuses are consistent across hourly rates (16-29), showing little variability.
- **Insights:**
  - Bonuses may depend more on performance or role-based incentives than hourly rates.
  - Equitable bonus distribution across pay grades is evident.

### 6. Hourly Rate Distribution Across Departments (Bar Chart)
- **Observation:** Departments 7, 6, and 3 have the highest average hourly rates.
- **Insights:**
  - Higher hourly rates in these departments suggest critical or senior roles.
  - Uniform distribution in other departments reflects standardized pay scales.

### 7. Age Distribution of Employees (Bar Chart)
- **Observation:** Employees aged 38 are the largest group, followed by those aged 39 and 37.
- **Insights:**
  - Mid-career professionals dominate the workforce, contributing expertise and stability.

### 8. Gender-wise Comparison of Hourly Rates (Bar Chart)
- **Observation:** Female employees have slightly higher average hourly rates than male employees.
- **Insights:**
  - Females might occupy more senior or higher-paying roles in the organization.
  - Gender-based role specialization could explain this disparity.

### 9. Employee Count by Department ID (Bar Chart)
- **Observation:** Department 7 has the largest workforce, followed by departments 6 and 3.
- **Insights:**
  - Department 7 likely represents a core organizational function.
  - Departments 6 and 3 also play vital roles with significant workforce allocations.

### 10. Lab-wise Bonus Allocation (Line Chart)
- **Observation:** Labs 2 and 9 lead in average bonuses.
- **Insights:**
  - Labs 2 and 9 house performance-driven roles with high incentives.
  - Uniform bonus levels in other labs suggest a balanced reward system.

---

## Managerial Insights

### 1. Workforce Demographics and Structure
- **Age Distribution:** Employees aged 37-39 dominate the workforce.
- **Gender Representation:** Female employees earn higher average hourly rates, reflecting equitable opportunities.

### 2. Compensation Patterns
- **Hourly Rates:** Departments 7, 6, and 3, along with Labs 9, 2, and 5, have the highest rates.
- **Bonuses:** Bonuses are equitably distributed, focusing on performance-based rewards.

### 3. Departmental Insights
- **Strategic Departments:** Department 7 is critical, with the largest workforce and highest average hourly rates.
- **Standardized Pay:** Uniform pay scales ensure fairness within departments.

### 4. Lab Performance and Allocations
- **High-Performing Labs:** Labs 2 and 9 excel in hourly rates and bonuses.
- **Consistent Pay:** Other labs maintain standardized pay structures.

### 5. Correlation Insights
- **Bonuses vs. Hourly Rates:** Bonuses are performance-driven, not directly tied to hourly rates.
- **Department Prioritization:** Higher hourly rates in specific departments highlight strategic importance.

### 6. Employee Engagement and Retention
- **Retention Focus:** Mid-career professionals require tailored retention strategies.
- **Performance Incentives:** Equitable bonus distribution fosters a performance-driven culture.

### 7. Potential Optimization Areas
- **Bonus Alignment:** Linking bonuses more closely with hourly rates could improve motivation.
- **Underperforming Labs:** Identify and improve labs with lower rates and bonuses.
- **Age Diversity:** Attracting younger employees (under 35) can enhance the talent pipeline.

### 8. Strategic Focus
- Prioritize investment in Departments 7, 6, and 3, along with Labs 2 and 9, to maximize organizational performance.

---

## Key Learnings
- **Equitable Compensation:** Bonuses and pay scales must balance equity and performance.
- **Strategic Resource Allocation:** Focus on high-performing departments and labs ensures efficiency.
- **Continuous Monitoring:** Regular adjustments to workforce demographics, pay scales, and bonuses align with organizational goals.

---
