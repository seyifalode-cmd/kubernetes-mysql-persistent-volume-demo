# Storage for mySQL pod in Kubernetes

1. Create an instance of a mySQL container
```bash
kubectl apply -f mysql-pod.yml 
```

2. Run two terminal windows, in the first window run this command.
```bash
kubectl get pod mysql --watch
```

3. In the second window, run this command to access the running container.
```bash
kubectl exec -it mysql -- /bin/bash
```

4. Using the interactive shell, connect to the database using the password provided in the mysql-pod.yml.
```bash
mysql -u root -p
```

5. Create the database, table and data using the SQL commands in the createData.sql file.

6. Exit the mysql window and kill the mysql process.
```bash
kill 1
```

7. Observe the restart of the pod in the first window.  Connect again to the pod using the second window.
```bash
kubectl exec -it mysql -- /bin/bash
```

8. Connect to the database using the password provided in the mysql-pod.yml.
```bash
mysql -u root -p
```

9. Switch to the YorkU database and select data from the Customer table.
```bash
USE YORKU;
SELECT * FROM CUSTOMERS;
```