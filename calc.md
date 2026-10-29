# Question 5 - Simple Banking Application with Kubernetes CI/CD

## Required outcome

- **Push 1:** Check balance and deposit money.
- **Push 2:** Add withdrawal with balance validation.
- Each push runs GitHub Actions tests and publishes a Docker image with that commit's unique SHA tag.
- Deploy the matching image to Kubernetes after each successful pipeline run.

> For the local Docker Desktop Kubernetes setup in the reference guide, GitHub-hosted Actions cannot directly reach the cluster on your laptop. Therefore, each push automatically tests and publishes its image; run the listed `kubectl` commands after each successful run to deploy the matching image. Do not make any extra Git pushes.

## 1. Check prerequisites

Open **VS Code → Terminal → New Terminal**. Use PowerShell and run each command:

```powershell
git --version
python --version
docker --version
docker info
kubectl version --client
kubectl config current-context
kubectl get nodes
```

Confirm Docker Desktop is running and Kubernetes is enabled. `kubectl get nodes` must show a node with status `Ready`.

## 2. Create the project

Run these commands in the VS Code terminal:

```powershell
mkdir student-k8s-demo
cd student-k8s-demo
mkdir k8s
mkdir -Force .github\workflows
code .
```

Create this file structure in VS Code:

```text
student-k8s-demo/
├── app.py
├── requirements.txt
├── test_app.py
├── Dockerfile
├── .dockerignore
├── .gitignore
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
└── .github/
    └── workflows/
        └── pipeline.yml
```

## 3. Create the Push 1 application

### 3.1 `app.py`

Paste this code. This version supports balance checking and deposits only.

```python
import math
from flask import Flask, jsonify, render_template_string, request

app = Flask(__name__)
balance = 1000.0

PAGE = """
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title>Simple Banking Application</title>
</head>
<body>
    <h1>Simple Banking Application</h1>
    <p>Current balance: {{ "%.2f"|format(balance) }}</p>
    <form method="post">
        <label for="action">Choose operation</label>
        <select name="action" id="action">
            <option value="balance">Check balance</option>
            <option value="deposit">Deposit money</option>
        </select>
        <label for="amount">Amount</label>
        <input type="number" name="amount" id="amount" min="0.01" step="0.01">
        <button type="submit">Submit</button>
    </form>
    <p>{{ message }}</p>
</body>
</html>
"""


@app.route("/", methods=["GET", "POST"])
def home():
    global balance
    message = ""
    if request.method == "POST":
        action = request.form.get("action", "")
        if action == "balance":
            message = f"Balance checked: {balance:.2f}"
        elif action == "deposit":
            try:
                amount = float(request.form.get("amount", "0"))
            except ValueError:
                amount = 0
            if not math.isfinite(amount) or amount <= 0:
                message = "Enter a deposit amount greater than zero."
            else:
                balance += amount
                message = f"Deposit successful. Current balance: {balance:.2f}"
        else:
            message = "Invalid option."
    return render_template_string(PAGE, balance=balance, message=message)


@app.route("/health")
def health():
    return jsonify(status="healthy"), 200


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

### 3.2 `requirements.txt`

```text
Flask>=3.0,<4.0
pytest>=8.0,<9.0
```

### 3.3 `test_app.py`

These are the Push 1 tests. Do not add withdrawal tests until Push 2.

```python
import app as banking


def setup_function():
    banking.balance = 1000.0


def test_home_page():
    response = banking.app.test_client().get("/")
    assert response.status_code == 200
    assert b"Simple Banking Application" in response.data


def test_health_endpoint():
    response = banking.app.test_client().get("/health")
    assert response.status_code == 200
    assert response.json["status"] == "healthy"


def test_balance_check():
    response = banking.app.test_client().post("/", data={"action": "balance"})
    assert response.status_code == 200
    assert b"Balance checked: 1000.00" in response.data


def test_deposit():
    response = banking.app.test_client().post(
        "/", data={"action": "deposit", "amount": "250"}
    )
    assert response.status_code == 200
    assert b"Deposit successful. Current balance: 1250.00" in response.data
    assert banking.balance == 1250.0


def test_invalid_deposit():
    response = banking.app.test_client().post(
        "/", data={"action": "deposit", "amount": "0"}
    )
    assert b"Enter a deposit amount greater than zero." in response.data
    assert banking.balance == 1000.0
```

## 4. Create Docker and Git ignore files

### 4.1 `Dockerfile`

```dockerfile
FROM python:3.12-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
```

### 4.2 `.dockerignore`

```text
.venv/
__pycache__/
.pytest_cache/
.git/
```

### 4.3 `.gitignore`

```text
.venv/
__pycache__/
.pytest_cache/
.env
```

## 5. Create the GitHub Actions pipeline

### 5.1 `.github/workflows/pipeline.yml`

This workflow runs tests first. Only if tests pass on a push to `main` will it build and push the Docker image. The image tag is the full commit SHA, so each push has a unique image tag.

```yaml
name: Student Banking Kubernetes CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  test:
    name: Checkout and Test
    runs-on: ubuntu-latest
    steps:
      - name: Checkout source code
        uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
      - name: Run automated tests
        run: python -m pytest -v

  docker:
    name: Build and Push Docker Image
    needs: test
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout source code
        uses: actions/checkout@v4
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/student-k8s-demo:${{ github.sha }}
```

## 6. Create Kubernetes manifests

### 6.1 `k8s/deployment.yaml`

Replace the username and SHA placeholders with the values for the push you are deploying. Keep one replica because this learning application stores its balance in application memory; separate replicas would have separate balances.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: student-k8s-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: student-k8s-demo
  template:
    metadata:
      labels:
        app: student-k8s-demo
    spec:
      containers:
        - name: student-k8s-demo
          image: YOUR_DOCKERHUB_USERNAME/student-k8s-demo:REPLACE_WITH_COMMIT_SHA
          ports:
            - containerPort: 5000
          readinessProbe:
            httpGet:
              path: /health
              port: 5000
            initialDelaySeconds: 5
            periodSeconds: 5
```

### 6.2 `k8s/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: student-k8s-demo
spec:
  type: NodePort
  selector:
    app: student-k8s-demo
  ports:
    - port: 80
      targetPort: 5000
      nodePort: 30080
```

## 7. Test the application locally

Run these commands from the `student-k8s-demo` folder in PowerShell. This approach does not require activating the virtual environment.

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe -m pytest -v
```

Expected: all **5 tests pass**.

Run the web application:

```powershell
.\.venv\Scripts\python.exe app.py
```

Open these addresses in your browser:

- `http://127.0.0.1:5000/`
- `http://127.0.0.1:5000/health`

The health endpoint should return `{"status":"healthy"}`. Stop Flask with `Ctrl+C` before starting the Docker container.

## 8. Build and test the Docker image locally

```powershell
docker build -t student-k8s-demo:local .
docker run --rm -d --name student-demo -p 5001:5000 student-k8s-demo:local
docker ps
docker logs student-demo
```

Open `http://localhost:5001/`. When finished, stop the container:

```powershell
docker stop student-demo
```

## 9. Prepare GitHub and Docker Hub before Push 1

1. Create a **public** GitHub repository named `student-k8s-demo`. Do not initialize it with a README or other files.
2. Create a **public** Docker Hub repository named `student-k8s-demo`.
3. In Docker Hub, create an access token with permission to push images. Copy the token somewhere safe temporarily.
4. In GitHub, open **Settings → Secrets and variables → Actions → New repository secret**.
5. Add these two secrets:

| Secret name | Value |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | The Docker Hub access token |

Do not put the token in any source file.

## 10. Push 1 - balance checking and deposits

Ensure `app.py`, `test_app.py`, all Docker/Kubernetes files, and `pipeline.yml` are saved before continuing.

Run these commands from the project directory. Replace `YOUR_GITHUB_USERNAME` with your GitHub username.

```powershell
git init
git add .
git commit -m "Push 1: Add balance and deposit features"
git branch -M main
git remote add origin https://github.com/YOUR_GITHUB_USERNAME/student-k8s-demo.git
git push -u origin main
```

This is **Push 1**. Do not make another Git push at this stage.

### 10.1 Verify Push 1 in GitHub Actions

1. Open your GitHub repository in a browser.
2. Select **Actions**.
3. Open the `Student Banking Kubernetes CI/CD` run for Push 1.
4. Confirm `Checkout and Test` succeeds.
5. Confirm `Build and Push Docker Image` succeeds.
6. Open Docker Hub and confirm that the image tag matching the pushed commit SHA exists.

Get the SHA for Push 1 in the VS Code terminal:

```powershell
git rev-parse HEAD
```

Copy the full SHA. The image name for Push 1 is:

```text
YOUR_DOCKERHUB_USERNAME/student-k8s-demo:YOUR_PUSH1_COMMIT_SHA
```

### 10.2 Deploy the Push 1 image to Kubernetes

In `k8s/deployment.yaml`, replace `YOUR_DOCKERHUB_USERNAME` with your Docker Hub username and `REPLACE_WITH_COMMIT_SHA` with the full Push 1 SHA. Save the file.

Check the cluster and deploy:

```powershell
kubectl config current-context
kubectl get nodes
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl rollout status deployment/student-k8s-demo
kubectl get deployments
kubectl get pods
kubectl get services
```

Open `http://localhost:30080/`. Check the balance and deposit a positive amount. Confirm that the displayed balance changes. Check `http://localhost:30080/health` and confirm that it reports a healthy status.

## 11. Prepare Push 2 - add withdrawals

Do not change the GitHub Actions workflow. Modify only `app.py` and `test_app.py` for Push 2.

### 11.1 Update `app.py`

Replace the contents of `app.py` with the following updated version. It includes balance checking, deposits, and withdrawals with balance validation.

```python
import math
from flask import Flask, jsonify, render_template_string, request

app = Flask(__name__)
balance = 1000.0

PAGE = """
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title>Simple Banking Application</title>
</head>
<body>
    <h1>Simple Banking Application</h1>
    <p>Current balance: {{ "%.2f"|format(balance) }}</p>
    <form method="post">
        <label for="action">Choose operation</label>
        <select name="action" id="action">
            <option value="balance">Check balance</option>
            <option value="deposit">Deposit money</option>
            <option value="withdraw">Withdraw money</option>
        </select>
        <label for="amount">Amount</label>
        <input type="number" name="amount" id="amount" min="0.01" step="0.01">
        <button type="submit">Submit</button>
    </form>
    <p>{{ message }}</p>
</body>
</html>
"""


@app.route("/", methods=["GET", "POST"])
def home():
    global balance
    message = ""
    if request.method == "POST":
        action = request.form.get("action", "")
        if action == "balance":
            message = f"Balance checked: {balance:.2f}"
        elif action == "deposit":
            try:
                amount = float(request.form.get("amount", "0"))
            except ValueError:
                amount = 0
            if not math.isfinite(amount) or amount <= 0:
                message = "Enter a deposit amount greater than zero."
            else:
                balance += amount
                message = f"Deposit successful. Current balance: {balance:.2f}"
        elif action == "withdraw":
            try:
                amount = float(request.form.get("amount", "0"))
            except ValueError:
                amount = 0
            if not math.isfinite(amount) or amount <= 0:
                message = "Enter a withdrawal amount greater than zero."
            elif amount > balance:
                message = "Insufficient balance. Withdrawal denied."
            else:
                balance -= amount
                message = f"Withdrawal successful. Current balance: {balance:.2f}"
        else:
            message = "Invalid option."
    return render_template_string(PAGE, balance=balance, message=message)


@app.route("/health")
def health():
    return jsonify(status="healthy"), 200


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

### 11.2 Update `test_app.py`

Replace the contents of `test_app.py` with the following tests for both Push 1 and Push 2 functionality.

```python
import app as banking


def setup_function():
    banking.balance = 1000.0


def test_home_page():
    response = banking.app.test_client().get("/")
    assert response.status_code == 200
    assert b"Simple Banking Application" in response.data
    assert b"Withdraw money" in response.data


def test_health_endpoint():
    response = banking.app.test_client().get("/health")
    assert response.status_code == 200
    assert response.json["status"] == "healthy"


def test_balance_check():
    response = banking.app.test_client().post("/", data={"action": "balance"})
    assert response.status_code == 200
    assert b"Balance checked: 1000.00" in response.data


def test_deposit():
    response = banking.app.test_client().post(
        "/", data={"action": "deposit", "amount": "250"}
    )
    assert response.status_code == 200
    assert b"Deposit successful. Current balance: 1250.00" in response.data
    assert banking.balance == 1250.0


def test_invalid_deposit():
    response = banking.app.test_client().post(
        "/", data={"action": "deposit", "amount": "0"}
    )
    assert b"Enter a deposit amount greater than zero." in response.data
    assert banking.balance == 1000.0


def test_withdrawal():
    response = banking.app.test_client().post(
        "/", data={"action": "withdraw", "amount": "200"}
    )
    assert response.status_code == 200
    assert b"Withdrawal successful. Current balance: 800.00" in response.data
    assert banking.balance == 800.0


def test_withdrawal_rejects_amount_greater_than_balance():
    response = banking.app.test_client().post(
        "/", data={"action": "withdraw", "amount": "1200"}
    )
    assert b"Insufficient balance. Withdrawal denied." in response.data
    assert banking.balance == 1000.0


def test_invalid_withdrawal():
    response = banking.app.test_client().post(
        "/", data={"action": "withdraw", "amount": "0"}
    )
    assert b"Enter a withdrawal amount greater than zero." in response.data
    assert banking.balance == 1000.0
```

### 11.3 Test Push 2 locally before pushing

```powershell
.\.venv\Scripts\python.exe -m pytest -v
```

Expected: all **8 tests pass**.

## 12. Push 2 - updated banking application

Run the following commands. Stage only `app.py` and `test_app.py`; do not stage `k8s/deployment.yaml`, because its image SHA is updated locally for deployment and must not be included in this commit.

```powershell
git add app.py test_app.py
git commit -m "Push 2: Add withdrawal with balance validation"
git push
```

This is **Push 2**. Do not make any further Git pushes for this assessment.

### 12.1 Verify Push 2 in GitHub Actions

1. Open **GitHub → your repository → Actions**.
2. Open the run caused by Push 2.
3. Confirm all tests pass.
4. Confirm `Build and Push Docker Image` succeeds.
5. In Docker Hub, confirm the image tag for the Push 2 commit SHA exists.

Get the new SHA:

```powershell
git rev-parse HEAD
```

The Push 2 image name is:

```text
YOUR_DOCKERHUB_USERNAME/student-k8s-demo:YOUR_PUSH2_COMMIT_SHA
```

## 13. Deploy the Push 2 image to Kubernetes

Update the image line in your local `k8s/deployment.yaml` to use the Push 2 SHA. Save the file, then run:

```powershell
kubectl apply -f k8s/deployment.yaml
kubectl rollout status deployment/student-k8s-demo
kubectl get deployments
kubectl get pods
kubectl get services
```

Alternatively, update the existing Deployment directly by replacing the username and SHA below:

```powershell
kubectl set image deployment/student-k8s-demo student-k8s-demo=YOUR_DOCKERHUB_USERNAME/student-k8s-demo:YOUR_PUSH2_COMMIT_SHA
kubectl rollout status deployment/student-k8s-demo
kubectl get pods
```

Open `http://localhost:30080/` and test all three options:

1. **Check balance:** the starting balance is `1000.00` after a fresh Pod starts.
2. **Deposit money:** deposit a positive amount and verify the balance increases.
3. **Withdraw money:** withdraw an amount within the balance and verify the balance decreases.
4. **Insufficient balance:** try withdrawing more than the current balance and confirm `Insufficient balance. Withdrawal denied.` appears and the balance does not decrease.

Check `http://localhost:30080/health` and confirm it reports `{"status":"healthy"}`.

## 14. Useful commands if something fails

```powershell
kubectl get all
kubectl describe deployment student-k8s-demo
kubectl get pods
kubectl logs deployment/student-k8s-demo
kubectl get events --sort-by=.metadata.creationTimestamp
```

- If Actions tests fail, open the failed step and check the error before proceeding.
- If Docker Hub login fails, verify `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` in GitHub repository secrets.
- If a Pod shows `ImagePullBackOff`, verify that the Docker Hub repository is public and the deployment image name and full SHA tag match the successful Actions run.
- If the browser cannot connect, check Pod readiness, Service status, and port `30080`.

## 15. Final assessment checklist

- [ ] Local Push 1 application and tests pass.
- [ ] Push 1 is the first and only initial Git push.
- [ ] GitHub Actions tests pass for Push 1.
- [ ] Push 1 Docker image is published with its commit SHA tag.
- [ ] Push 1 image is deployed to Kubernetes and the app is accessible.
- [ ] Push 2 adds withdrawal and balance validation.
- [ ] Push 2 is the second and final Git push.
- [ ] GitHub Actions tests pass for Push 2.
- [ ] Push 2 Docker image is published with a different commit SHA tag.
- [ ] Kubernetes is updated to the Push 2 image.
- [ ] Balance checking, deposits, successful withdrawals, and insufficient-balance validation work in the browser.
