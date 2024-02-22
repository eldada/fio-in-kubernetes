# Disk IO tests with fio in Kubernetes
This repository has a set of tools to build and optionally run [fio](https://fio.readthedocs.io/en/latest/fio_doc.html) in Kubernetes pods

## Get The Binary
You can skip the build and download the pre-built binary directly from
- x86_64
```shell
# Get file and make it executable
curl -L https://eldada.jfrog.io/artifactory/tools/fio/3.36/fio-linux-x86_64 -o ./fio
chmod +x fio

# Test it works
./fio --version
```

- ARM
```shell
# Get file and make it executable
curl -L https://eldada.jfrog.io/artifactory/tools/fio/3.36/fio-linux-arm64 -o ./fio
chmod +x fio

# Test it works
./fio --version
```

## Build of Install fio in a Kubernetes pod
### Build fio from sources
Building `fio` requires the same processor architecture of the server you'll be running the tests on.</br>
This repository includes two pods templates that deploy to the current Kubernetes cluster and are assigned to a node with the desired architecture.</br>
The pod comes up and
1. Installs the needed tools
2. Clones the official `fio` sources (https://github.com/axboe/fio)
3. Compiles and builds `fio` (with `--build-static` so you have the dependencies built into the binary)

Once built, you can then copy the `fio` binary from the pod and take it anywhere. 

### Install fio
If you just want `fio` in the pod, you can just set the environment variable `BUILD_OR_INSTALL` in the yamls to `install` and deploy.</br>
This will install `fio` with `apt`.

Deploy the pods with the following commands.<br>
Make sure to set affinity and toleration rules as needed to make sure you get the right architecture binary built.
```shell
export NAMESPACE=0-fio-build

kubectl create namespace ${NAMESPACE}

kubectl apply -n ${NAMESPACE} -f fio-x86_64.yaml

kubectl apply -n ${NAMESPACE} -f fio-arm64.yaml
```

Follow the pods logs to see the progress and get the command to copy the binary to your computer if needed
```shell
kubectl logs -n ${NAMESPACE} pod-fio-x86-64 -f

kubectl logs -n ${NAMESPACE} pod-fio-arm64 -f
```

## Run the IO tests in the pods
You can run `fio` directly in the pods and get the Kubernetes node's local disk tested using the pods that were just deployed.</br>
You can test an externally mounted volume be setting the `--filename` parameter to the mounted volume in the pod.

See the [official fio website](https://fio.readthedocs.io/en/latest/fio_doc.html) for all the available options and parameters.

The examples below runs in the running `x86_64` pod. It uses json output, 10 concurrent jobs, 1mb block size, and saves the output in a local file for analysis later
```shell
mkdir -p out

# Sequential reads for 30 seconds
kubectl exec -n ${NAMESPACE} pod-fio-x86-64 -- fio --name=test --output-format=json --filename=/tmp/test.fio --size=2g --runtime=30s --ioengine=libaio --rw=read --direct=1 --numjobs=10 --blocksize=1m > out/read.json

# Sequential writes for 30 seconds
kubectl exec -n ${NAMESPACE} pod-fio-x86-64 -- fio --name=test --output-format=json --filename=/tmp/test.fio --size=2g --runtime=30s --ioengine=libaio --rw=write --direct=1 --numjobs=10 --blocksize=1m > out/write.json

# Random read/write for 30 seconds
kubectl exec -n ${NAMESPACE} pod-fio-x86-64 -- fio --name=test --output-format=json --filename=/tmp/test.fio --size=2g --runtime=30s --ioengine=libaio --rw=randrw --direct=1 --numjobs=10 --blocksize=1m > out/randrw.json
```

Extract the MBPS (MegaBytes per seconds) from the output json (using `jq`)
```shell
# Reads average MBPS from the 10 jobs results
jq '[.jobs[] | .read.io_bytes] | add/1024/1024' out/read.json

# Writes average MBPS from the 10 jobs results
jq '[.jobs[] | .write.io_bytes] | add/1024/1024' out/write.json

# Reads average MBPS from the 10 jobs results out of the RW tests
jq '[.jobs[] | .read.io_bytes] | add/1024/1024' out/randrw.json

# Writes average MBPS from the 10 jobs results out of the RW tests
jq '[.jobs[] | .write.io_bytes] | add/1024/1024' out/randrw.json
```
