# Readme vulnerability scan

```bash
docker run -d --name db arminc/clair-db:latest
docker run -p 6060:6060 --link db:postgres -d --name clair --restart on-failure docker.digilab.ocpgroup.ma/devops/clair-local-scan:v2.0.1
```

```bash
wget https://github.com/arminc/clair-scanner/releases/download/v12/clair-scanner_darwin_amd64
mv clair-scanner_darwin_amd64 clair-scanner
chmod +x clair-scanner
```

To showcase the basic usage of container scaning, let's scan an official image of nginx: nginx:1.14

```bash
./clair-scanner -c http://127.0.0.1:6060  --ip host.docker.internal  -r gl-container-scanning-report.json -l clair.log nginx:1.14.0
```

The report shows  **153 vulnerabilities**

```bash
2019/10/28 16:39:38 [WARN] ▶ Image [nginx:1.14.0] contains 153 total vulnerabilities
2019/10/28 16:39:38 [ERRO] ▶ Image [nginx:1.14.0] contains 153 unapproved vulnerabilities
```

Let scan now the alpine distribution of nginx. It's a small distribution, it’s focused on security and exposes a very small attack surface.

```bash
./clair-scanner -c http://127.0.0.1:6060  --ip host.docker.internal  -r gl-container-scanning-report-2.json -l clair-2.log nginx:1.14-alpine
```

BOOOOM, only **7 vulnerabilities** are identified.

```text
2019/10/28 16:40:56 [WARN] ▶ Image [nginx:1.14-alpine] contains 7 total vulnerabilities
2019/10/28 16:40:56 [ERRO] ▶ Image [nginx:1.14-alpine] contains 7 unapproved vulnerabilities
```

So you see now the impact of well choosing the base image for your build images.
