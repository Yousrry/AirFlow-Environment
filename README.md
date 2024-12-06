# **Apache Airflow Environment with Docker Compose**

This project sets up an **Apache Airflow environment** using Docker Compose. It includes the following components:
- **Airflow Webserver**: User interface for managing workflows.
- **Airflow Scheduler**: Schedules and monitors task execution.
- **Celery Workers**: Distributed workers for executing tasks (3 replicas).
- **PostgreSQL**: Metadata database.
- **Redis**: Message broker for task queuing.

## **Prerequisites**
Before running this setup, ensure you have:
1. **Docker** installed ([Installation guide](https://docs.docker.com/get-docker/)).
2. **Docker Compose** installed ([Installation guide](https://docs.docker.com/compose/install/)).

---

## **How to Use**

### **1. Initialize the Airflow Metadata Database**
Run the following command to initialize the Airflow metadata database and set up the required tables:
```bash
docker-compose up airflow-init
```

### **2. Start the Airflow Environment**
Start all services in detached mode:
```bash
docker-compose up -d
```

### **3. Access the Airflow Web Interface**
Open your browser and go to [http://localhost:8080](http://localhost:8080).  
Default login credentials:
- **Username**: `admin`
- **Password**: `admin` (can be changed via the UI)

### **4. Stop the Services**
To stop the containers, run:
```bash
docker-compose down
```

---

## **Project Structure**

### **Docker Compose Configuration**
The `docker-compose.yaml` is structured as follows:

### **PostgreSQL**
- **Purpose**: Metadata database for Airflow.
- **Key Features**:
  - Stores Airflow’s workflows, task instances, and execution history.
  - Configured with a named volume to persist data.

### **Redis**
- **Purpose**: Acts as a message broker for Celery Executor.
- **Key Features**:
  - Queues tasks for workers.
  - Lightweight and fast in-memory database.

### **Airflow Scheduler**
- **Purpose**: Schedules tasks and monitors execution.
- **Key Features**:
  - Connects to PostgreSQL for metadata.
  - Relies on Redis for queuing tasks.
  - Logs stored persistently for debugging.

### **Airflow Webserver**
- **Purpose**: User interface for managing workflows and configurations.
- **Key Features**:
  - Exposes port `8080` for web access.
  - Shows task logs for debugging.

### **Celery Workers**
- **Purpose**: Executes tasks distributed by the Scheduler.
- **Key Features**:
  - Scalable with 3 replicas configured.
  - Logs execution results in the PostgreSQL backend.

---

## **Docker Compose File Explanation**

```yaml
version: '3.8'
```
- Specifies the version of the Docker Compose specification.

### **PostgreSQL Database**
```yaml
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - airflow_network
```
- Configures PostgreSQL with:
  - User: `airflow`
  - Password: `airflow`
  - Database: `airflow`
- Persists data using the `postgres_data` volume.
- Connects to the `airflow_network`.

---

### **Redis (Message Broker)**
```yaml
  redis:
    image: redis:7
    networks:
      - airflow_network
```
- Configures Redis as the message broker for Celery Executor.
- Connects to the `airflow_network`.

---

### **Airflow Scheduler**
```yaml
  scheduler:
    image: apache/airflow:2.7.2
    command: scheduler
    environment:
      AIRFLOW__CORE__EXECUTOR: CeleryExecutor
      AIRFLOW__CORE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
      AIRFLOW__CELERY__BROKER_URL: redis://redis:6379/0
      AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
    volumes:
      - airflow_logs:/opt/airflow/logs
      - .:/opt/airflow/dags
    depends_on:
      - postgres
      - redis
    networks:
      - airflow_network
```
- Configures the Scheduler to:
  - Use the **Celery Executor**.
  - Connect to PostgreSQL for metadata storage.
  - Use Redis for task queuing.
  - Store logs persistently in the `airflow_logs` volume.

---

### **Airflow Webserver**
```yaml
  webserver:
    image: apache/airflow:2.7.2
    command: webserver
    environment:
      AIRFLOW__CORE__EXECUTOR: CeleryExecutor
      AIRFLOW__CORE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
      AIRFLOW__CELERY__BROKER_URL: redis://redis:6379/0
      AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
    ports:
      - "8080:8080"
    volumes:
      - airflow_logs:/opt/airflow/logs
      - .:/opt/airflow/dags
    depends_on:
      - scheduler
    networks:
      - airflow_network
```
- Configures the Webserver to:
  - Run on port `8080`.
  - Connect to the Scheduler, PostgreSQL, and Redis.

---

### **Celery Workers**
```yaml
  worker:
    image: apache/airflow:2.7.2
    command: celery worker
    environment:
      AIRFLOW__CORE__EXECUTOR: CeleryExecutor
      AIRFLOW__CORE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
      AIRFLOW__CELERY__BROKER_URL: redis://redis:6379/0
      AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
    volumes:
      - airflow_logs:/opt/airflow/logs
      - .:/opt/airflow/dags
    depends_on:
      - scheduler
    networks:
      - airflow_network
    deploy:
      replicas: 2
```
- Configures Celery Workers to:
  - Use 2 replicas for scalability.
  - Execute tasks distributed by the Scheduler.

---

### **Volumes**
```yaml
volumes:
  postgres_data:
  airflow_logs:
```
- **postgres_data**: Persistent storage for PostgreSQL database.
- **airflow_logs**: Persistent storage for Airflow logs.

---

### **Networks**
```yaml
networks:
  airflow_network:
```
- Creates a private network for secure inter-container communication.

---

## **Best Practices Followed**
1. **Modularity**: Each service is independent for scalability and maintainability.
2. **Persistence**: Logs and metadata are stored using Docker volumes.
3. **Isolation**: A custom network (`airflow_network`) isolates the environment.
4. **Scalability**: Celery workers are replicated (3 replicas) to handle high workloads.
5. **Dynamic Configuration**: Environment variables are used for flexibility.

---

## **Additional Commands**


### Scale Workers:
To scale the number of Celery workers:
```bash
docker-compose up --scale worker=N
```
Replace `N` with the desired number of workers.

---
## Docker compose up output 
![alt text](Images/image.png)
## Screenshots From the executed DAG !
![alt text](Images/image-1.png)
![alt text](Images/image-2.png)
