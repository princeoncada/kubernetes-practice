# Project Overview

This repository contains a multi-container application setup with Vite + React, Node.js, and MySQL, orchestrated with Kubernetes. It provides a scalable infrastructure that facilitates both development and production environments.

### Structure
- **client**: Contains the frontend React application setup using Vite.
- **server**: Houses the backend API built with Node.js.
- **k8s**: Stores all Kubernetes manifest files for deployment and service configuration.

### Technology Stack
- **Frontend**: Vite + React
- **Server**: Node.js with Express
- **Database**: MySQL
- **Orchestration**: Kubernetes
- **CI/CD**: Skaffold ( _for local development_ )

### Prerequisites
- [**Docker**](https://www.docker.com/products/docker-desktop/) with Kubernetes enabled
- [**Chocolatey**](https://chocolatey.org/install) (for Windows users)
- [**kubectl**](https://kubernetes.io/docs/tasks/tools/)
- [**Helm**](https://helm.sh/docs/intro/install/)
- [**Skaffold**](https://skaffold.dev/docs/install/)

<br></br>
# Inital Project Files

### Directory Structure
Create the main project directory and navigate into it:
```
mkdir my-project
cd my-project
```
Within this directory, we will create three subdirectories for our client, API server, and Nginx web server:

```
mkdir client
mkdir server
mkdir k8s
```
## Setting up the Client
1. **Create the Vite + React application:**
```
npm create vite@latest client -- --template react
cd client
```
2. **Install dependencies:**
```
npm install
npm install --save-dev vitest jsdom @testing-library/jest-dom @testing-library/react @testing-library/user-event axios
```
3. **Configure the development server in `vite.config.js`:**

    Add the server configuration block to the file.
```
export default defineConfig({
    // other configurations
    server: {
        host: '0.0.0.0',
        port: 3000
    },
});

```
4. **Replace `App.jsx` contents with the following code:**
```
import { useState, useEffect } from 'react'
import axios from 'axios';
import './App.css'

function App() {
    const [response, setResponse] = useState([
        {
            id: 1,
            data: "default data #1"
        },
        {
            id: 2,
            data: "default data #2"
        },
        {
            id: 3,
            data: "default data #3"
        }
    ]) 
    const [counter, setCounter] = useState(0)

    // Function to fetch data from the server
    function fetchData() {
        axios.get('/api/data')
            .then((response) => {
                setResponse(response.data);
            })
            .catch((error) => {
                console.error('Error fetching data:', error);
            });
    }

    useEffect(() => {
        fetchData();
    }, []);

    return (
        <>
            <button onClick={() => setCounter((counter) => counter + 1)}>
                {response[counter%response.length].data}
            </button>
        </>
    )
}

export default App
```
5. **Setup Testing Environment:** ( *Vite + React doesn't have a default testing setup* )
>- Inside `package.json` include in scripts this code:
```
"test": "vitest run"
```
>- Create a directory called `tests` inside the `client` directory.
>- Inside `tests`, create a file called `setup.js`.
>- Add the following code into the file:
```
import { afterEach } from 'vitest'
import { cleanup } from '@testing-library/react'

afterEach(() => {
    cleanup();
})
```
>- Add to `vite.config.js`:
```
export default defineConfig({
    // other configurations
    test: {
        environment: 'jsdom',
        globals: true,
        setupFiles: './tests/setup.js'
    }
});
```
>- Inside client `src` directory, add a file called `App.test.jsx`.
>- Add the following code into the file:
```
import { render, screen } from '@testing-library/react'
import App from './App'

describe('App', () => {
    it('renders the App component', () => {
        render(<App />)
        screen.debug();
    })
})
```
>- In your terminal, while inside client directory, run the test setup:
```
npm run test
```
7. Create a file called `Dockerfile.dev` inside the `client` directory.
8. Add the following code into the file:
```
FROM node:alpine
WORKDIR '/app'
COPY package.json .
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "run", "dev"]
```
## Setting Up the API Server
1. **Navigate to the server directory and initialize npm:**
```
cd ../server
npm init -y
```
2. **Install default packages:**
```
npm install express body-parser cors mysql2 nodemon
```
3. **Add scripts to package.json for development and start:**
```
"scripts": {
    // other scripts
    "dev": "nodemon index.js",
    "start": "node index.js"
}
```
4. **Include necessary files inside server directory:**
>- Create `keys.js` and add this code into it into the file:
```
module.exports = {
    mysqlUser: process.env.MYSQLUSER,
    mysqlHost: process.env.MYSQLHOST,
    mysqlDatabase: process.env.MYSQLDATABASE,
    mysqlPassword: process.env.MYSQLPASSWORD,
    mysqlPort: process.env.MYSQLPORT,
}
```
>- Create `index.js` and add this code into the file:
```
const keys = require('./keys');

// Express App Setup
const express = require('express');
const bodyParser = require('body-parser');
const cors = require('cors');

const app = express();
app.use(cors());
app.use(bodyParser.json());

// MySQL Client Setup
const mysql = require('mysql2/promise');

const pool = mysql.createPool({
    host: keys.mysqlHost,
    user: keys.mysqlUser,
    password: keys.mysqlPassword,
    database: keys.mysqlDatabase,
    port: keys.mysqlPort,
});

// MySQL Pool Connection Test
pool.getConnection((err, connection) => {
    if (err) {
        console.log('Error connecting to MySQL: ', err);
    } else {
        console.log('Connected to MySQL');
    }
});

// MySQL low-level migration
pool
    .query(
        `CREATE TABLE IF NOT EXISTS tbl_test (
        id INT AUTO_INCREMENT PRIMARY KEY,
        data VARCHAR(255));`
    )
    .then(
        pool.query(
            `INSERT INTO tbl_test (data) VALUES ('data #1'), ('data #2'), ('data #3');`
        )
    )
    .catch((err) => {
        console.log('Error creating table: ', err);
    });

// ***********************
// QUERIES AND ROUTES HERE
// ***********************
app.get('/data', async (req, res) => {
    try {
        const [rows, fields] = await pool.query('SELECT * FROM tbl_test');
        res.json(rows);
    } catch (err) {
        console.log('Error querying data: ', err);
    }
});


// 5000 is default, you may change as needed
const PORT = process.env.PORT || 5000;
app.listen(PORT, () => {
    console.log(`Server listening on port ${PORT}`);
});
```
6. Create a file called `Dockerfile.dev` inside the `server` directory.
7. Add the following code into the file:
```
FROM node:alpine
WORKDIR '/usr/src/app'
COPY package.json .
RUN npm install
COPY . .
EXPOSE 5000
CMD ["npm", "run", "dev"]
```

# Local Development Setup with Skaffold
🚩 _At this point, please make sure Kubernetes, kubectl, and helm are readily available._

### 1. NGINX Ingress Controller Manifest
- Inside your CMD, run the following commands:
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx 
helm repo update 
helm install ingress-nginx ingress-nginx/ingress-nginx
```
- Create a file called `ingress-service.yaml` inside your `k8s` directory.
- Add the following code into the file:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
    name: ingress-service
    annotations:
    nginx.ingress.kubernetes.io/use-regex: 'true'
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
    ingressClassName: nginx
    rules:
    - http:
        paths:
        - path: /?(.*)
        pathType: ImplementationSpecific
        backend:
            service:
            name: client-cluster-ip-service # Comes from client-cluster-ip-service.yaml
            port:
                number: 3000
        - path: /api/?(.*)
        pathType: ImplementationSpecific
        backend:
            service:
            name: server-cluster-ip-service # Comes from server-cluster-ip-service.yaml
            port:
                number: 5000
```

### 2. Client Deployment and Service Manifests
- Create a file called `client-deployment.yaml` inside your `k8s` directory.
- Add the following code into the file:
```
apiVersion: apps/v1
kind: Deployment
metadata:
    name: client-deployment
spec:
    replicas: 1
    selector:
    matchLabels:
        component: client
    template:
    metadata:
        labels:
        component: client # Used in client-cluster-ip-service.yaml
    spec:
        containers:
        - name: client
            image: pgsoncada/client-image
            ports:
            - containerPort: 3000
```
- Create a file called `client-cluster-ip-service.yaml` inside your `k8s` directory.
- Add the following code into the file:
```
apiVersion: v1
kind: Service
metadata:
    name: client-cluster-ip-service # Used in ingress-service.yaml
spec:
    type: ClusterIP
    selector:
    component: client # Comes from client-deployment.yaml
    ports:
    - port: 3000
        targetPort: 3000
```

### 3. Server Deployment and Service Manifests
- Create a file called `server-deployment.yaml` inside your `k8s` directory.
- Add the following code into the file:
```
apiVersion: apps/v1
kind: Deployment
metadata:
    name: server-deployment
spec:
    replicas: 1
    selector:
    matchLabels:
        component: server
    template:
    metadata:
        labels:
        component: server # Used in server-cluster-ip-service.yaml
    spec:
        containers:
        - name: server
            image: pgsoncada/server-image
            ports:
            - containerPort: 5000
            env:
            - name: MYSQLUSER
                value: 'user'
            - name: MYSQLPASSWORD
                valueFrom:
                secretKeyRef:
                    name: db-secrets
                    key: db-password
            - name: MYSQLDATABASE
                value: 'db_test'
            - name: MYSQLHOST
                value: 'mysql-cluster-ip-service' # Comes from mysql-cluster-ip-service.yaml
            - name: MYSQLPORT
                value: '3306'
```
- Create a file called `server-cluster-ip-service.yaml` inside your `k8s` directory.
- Add the following code into the file:
```
apiVersion: v1
kind: Service
metadata:
    name: server-cluster-ip-service # Used in ingress-service.yaml
spec:
    type: ClusterIP
    selector:
    component: server # Comes from server-deployment.yaml
    ports:
    - port: 5000
        targetPort: 5000
```

### 4. Database (MySQL) Deployment, Service, and Persistent Volume Claim Manifests
- Create a file called `db-persistent-volume-claim.yaml ` inside your `k8s` directory.
- Add the following code into the file:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: db-persistent-volume-claim # Used in mysql-deployment.yaml
spec:
    accessModes:
    - ReadWriteOnce
    resources:
    requests:
        storage: 2Gi # Specify the size of the storage required
```
- Create a file called `mysql-deployment.yaml` inside your `k8s` directory.
- Add the following code into the file:
```
apiVersion: apps/v1
kind: Deployment
metadata:
    name: mysql-deployment
spec:
    replicas: 1
    selector:
    matchLabels:
        component: mysql
    template:
    metadata:
        labels:
        component: mysql # Used in mysql-cluster-ip-service.yaml
    spec:
        volumes:
        - name: mysql-storage
        persistentVolumeClaim:
            claimName: db-persistent-volume-claim # Comes from db-persistent-volume-claim.yaml
        containers:
        - name: mysql
        image: mysql:5.7 # Or whichever version you prefer
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
            valueFrom:
            secretKeyRef:
                name: db-secrets
                key: db-root-password
        - name: MYSQL_DATABASE
            value: 'db_test'
        - name: MYSQL_USER
            value: 'user'
        - name: MYSQL_PASSWORD
            valueFrom:
            secretKeyRef:
                name: db-secrets
                key: db-password
```
- Create a file called `mysql-cluster-ip-service.yaml` inside `k8s` directory.
- Add the following code into the file:
```
apiVersion: v1
kind: Service
metadata:
    name: mysql-cluster-ip-service # Used in server-deployment.yaml
spec:
    type: ClusterIP
    selector:
        component: mysql # Comes from mysql-deployment.yaml
    ports:
    - port: 3306
    targetPort: 3306
```

### 5. Skaffold Setup
- Inside your `Command Line Interface`, run the following commands: ( _make sure to set your own db-password and db-root-password_ )
```
kubectl create secret generic db-secrets --from-literal=db-password=YOUR_DB_PASSWORD --from-literal=db-root-password=YOUR_DB_ROOT_PASSWORD
```
- Inside your `Command Line Interface`, redirect into your `k8s` directory, then run the following commands:
```
kubectl apply -f db-persistent-volume-claim.yaml
kubectl apply -f mysql-deployment.yaml
kubectl apply -f mysql-cluster-ip-service.yaml
```
- Inside your root directory, create a file named `skaffold.yaml`.
- Add the following code into the file:
```
apiVersion: skaffold/v2beta12
kind: Config
deploy:
  kubectl:
    manifests:
      - ./k8s/client-deployment.yaml
      - ./k8s/server-deployment.yaml
      - ./k8s/client-cluster-ip-service.yaml
      - ./k8s/server-cluster-ip-service.yaml
      - ./k8s/ingress-service.yaml
build:
  local:
    push: false
  artifacts:
    - image: pgsoncada/client-image
      context: client
      docker:
        dockerfile: Dockerfile.dev
      sync:
        manual:
          - src: "src/**/*.js"
            dest: .
          - src: "src/**/*.css"
            dest: .
          - src: "src/**/*.html"
            dest: .
          - src: "src/**/*.jsx"
            dest: .
    - image: pgsoncada/server-image
      context: server
      docker:
        dockerfile: Dockerfile.dev
      sync:
        manual:
          - src: "*.js"
            dest: .
```
- To test if everything is working, go to your `CLI` and while inside your `root` directory, run the following command:
```
skaffold dev
```
<br></br>
# Production Setup with Google Cloud
To Follow...
