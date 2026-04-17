![[Pasted image 20260406172732.png]]

#### Install kubectl

1. Download latest release (Linux x86-64)

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

2. Install kubectl

```bash
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

3. test to installed

```bash
kubectl version --client
```

