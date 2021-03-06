# Sourcegraph Developer README

## Cutting a release

### Make the desired changes to this repository

#### Updating docker image tags

- The vast majority of the time, [Renovate](https://renovatebot.com/docs/docker/) will open PRs in a timely manner.

- If you want to update them manually, you can update the Docker image versions in `*.Deployment.yaml` to match the tagged version that you are releasing.

  - You should look at our [DockerHub repositories](https://hub.docker.com/r/sourcegraph/) to see what the latest versions are.

  - Make sure to include the sha256 digest for each image, which [ensures that each image pull is immutable](https://renovatebot.com/docs/docker/#digest-pinning). Use `docker inspect --format='{{index .RepoDigests 0}}' $IMAGE` to get the digest.

### Open a PR

Wait for buildkite to pass and for your changes to be approved, then merge and check out `master`.

### Smoke test

Test what is currently checked in to master by [installing](docs/install.md) Sourcegraph on a fresh cluster.

#### Provision a cluster and install Sourcegraph

##### Using the sourcegraph/deploy-k8s-helper tool

Clone https://github.com/sourcegraph/deploy-k8s-helper to your machine and follow the [README](https://github.com/sourcegraph/deploy-k8s-helper/blob/master/README.md) to set up all the prerequisistes. 

###### Do smoke tests for `master` branch

1. Ensure that the `deploySourcegraphRoot` value in your stack configuration (see https://github.com/sourcegraph/deploy-k8s-helper/blob/master/README.md) is pointing to your deploy-sourcegraph checkout (ex: `pulumi config set deploySourcegraphRoot /Users/ggilmore/dev/go/src/github.com/sourcegraph/deploy-sourcegraph`)
1. In your deploy-sourcegraph checkout, make sure that you're on  the latest `master`
1. Run `yarn up` in your https://github.com/sourcegraph/deploy-k8s-helper checkout 
1. It'll take a few minutes for the cluster to be provisioned and for sourcegraph to be installed. Pulumi will show you the progresss that it's making, and will tell you when it's done. 
1. Use the instructions in [configure.md](docs/configure.md) to:
   1. Add a repository (e.g. [sourcegraph/sourcegraph](https://github.com/sourcegraph/sourcegraph))
   1. Enable a language extension (e.g. [Go](https://sourcegraph.com/extensions/sourcegraph/lang-go)), and test that code intelligence is working on the above repository
   1. Do a few test searches
1. When you're done, run `yarn destroy` to tear the cluster down. This can take ~10 minutes.

###### Check the upgrade path from the previous release to `master`

1. In your deploy-sourcegraph checkout, checkout the commit that contains the configuration for the previous release (e.g. the commit that has `2.11.x` images if you're currently trying to release `2.12.x`, etc.)
1. Run `yarn up` in your https://github.com/sourcegraph/deploy-k8s-helper checkout 
1. Do [the same smoke tests that you did above](#Do-smoke-tests-for-master-branch)
1. In your deploy-sourcegraph checkout, checkout the latest `master` commit again and run `yarn up` to deploy the new images. Check to see that [the same smoke tests](#Do-smoke-tests-for-master-branch) pass after the upgrade process. 
1. When you're done, run `yarn destroy` to tear the cluster down.

##### Manual instructions

###### Provision a new cluster

1.  Create a cluster unique name that is identifiable to you (e.g `ggilmore-test`) in the [Sourcegraph Auxiliary GCP Project](https://console.cloud.google.com/kubernetes/list?project=sourcegraph-server&organizationId=1006954638239).
    - You can create a pool that has `4` nodes, each with `8` vCPUs and `30` GB memory (for a total of `32` vCPUs and `120` GB memory).
    - See this screenshot, but note that the UI could have changed: ![](https://imgur.com/RuCyGX2.png)
1.  It’ll take a few minutes for it to be provisioned. You’ll see a green checkmark when it is done.
1.  Click on the `connect` button next to your cluster, it’ll give you a command to copy+paste in your terminal.
1.  Run the command in your terminal. Once it finishes, run `kubectl config current-context`. It should tell you that it’s set up to talk to the cluster that you just created.

###### Do smoke tests for `master` branch

1. Deploy the latest `master` to your new cluster by running through the quickstart steps in [docs/install.md](docs/install.md)
   - You'll need to create a GCP Storage Class named `sourcegraph` with the same `zone` that you created your cluster in (see ["Configure a storage class"](docs/configure.md#Configure-a-storage-class))
   - In order to give yourself permissions to create roles on the cluster, run: `kubectl create clusterrolebinding cluster-admin-binding --clusterrole cluster-admin --user $YOUR_NAME@sourcegraph.com`
1. Use the instructions in [configure.md](docs/configure.md) to:
   1. Add a repository (e.g. [sourcegraph/sourcegraph](https://github.com/sourcegraph/sourcegraph))
   1. Enable a language extension (e.g. [Go](https://sourcegraph.com/extensions/sourcegraph/lang-go)), and test that code intelligence is working on the above repository
   1. Do a couple test searches

###### Check the upgrade path from the previous release to `master`

1. Tear down the cluster that you created above by deleting it through from the [Sourcegraph Auxiliary GCP Project](https://console.cloud.google.com/kubernetes/list?project=sourcegraph-server&organizationId=1006954638239).
1. Checkout the commit that contains the configuration for the previous release (e.g. the commit has `2.11.x` images if you're currently trying to release `2.12.x`, etc.)
1. [Use the "Provision a new cluster" instructions above](#Provision-a-new-cluster) to create a new cluster.
1. Deploy the older commit to the new cluster, and do [the same smoke tests](#Do-smoke-tests-for-master-branch) with the older version.
1. Checkout the latest `master`, deploy the newer images to the same cluster (without tearing it down in between) by running `./kubectl-apply-all.sh`, and check to see [that the smoke test](#Do-smoke-tests-for-master-branch) passes after the upgrade process.

### Tag the release

The version numbers for [sourcegraph/deploy-sourcegraph](https://github.com/sourcegraph/deploy-sourcegraph) largely follow [sourcegraph/sourcegraph](https://github.com/sourcegraph/sourcegraph)'s version numbers (i.e. `deploy-sourcegraph@v2.11.2` uses [sourcegraph/sourcegraph](https://github.com/sourcegraph/sourcegraph)'s `v2.11.2` image tags).

#### [sourcegraph/deploy-sourcegraph](https://github.com/sourcegraph/deploy-sourcegraph)'s branching strategy

##### Cutting a new minor version (e.g. `v2.12.0`)

- Make a branch named `2.12` that stems from the current `master` branch and push it. `2.12` will be used as the base for all `v2.12.x` images, and future changes to the `v2.12.x` series will be cherry-picked onto it and tagged from there.

  ```bash
  git checkout master
  git pull # make sure that you're up to date
  git checkout -b 2.12
  git push --set-upstream origin 2.12
  ```

- Follow [the tagging instructions below](#Create-a-git-tag-and-push-it-to-the-repository) to tag the `v2.12.0` release from the `2.12` branch.

##### Cutting a new patch version (e.g. `v2.12.1`)

- Cherry-pick the relevant commits from `master` that update the image tags to `v2.12.1` onto the `2.12` branch (which should have been created as mentioned above).

  ```bash
  git checkout 2.12
  git pull # make sure that you're up to date
  git cherry-pick commit0 commit1 ... # all relevant commits from the master branch
  git push
  ```

- Follow [the tagging instructions below](#Create-a-git-tag-and-push-it-to-the-repository) to tag the `v2.12.1` release from the `2.12` branch.

##### Only updating Kubernetes manifests/docs

_See [this commit](https://github.com/sourcegraph/deploy-sourcegraph/commit/1d1846f67c01ad2a81741cf95ee867405d3de3ab) as an example of a change that would fall into this category._

- Cherry-pick the relevant commits from `master` that update the manifests/docs onto the `2.12` branch.

  ```bash
  git checkout 2.12
  git pull # make sure that you're up to date
  git cherry-pick commit0 commit1 ... # all relevant commits from the master branch
  git push
  ```

- We mark these kinds of releases with a version number that looks like `v2.12.0-2`. Note the `-2` suffix, which increments with each update like this that affects the `v2.12.0` series. Look at [the releases page](https://github.com/sourcegraph/deploy-sourcegraph/releases), and pick the `suffix` that's next in the series.

- Follow [the tagging instructions below](#Create-a-git-tag-and-push-it-to-the-repository) to tag the `v2.12.0-2` release from the `2.12` branch.

#### Create a git tag and push it to the repository

```bash
VERSION = vX.Y.Z

# If this is a release candidate: VERSION = `vX.Y.Z-pre${N}` (where `N` starts at 0 and increments as you test/cut new versions)

# 🚨 Make sure that you have the commit that you want to tag as $VERSION checked out!

git tag $VERSION
git push origin $VERSION
```

### Update the `latestReleaseKubernetesBuild` value in `sourcegraph/sourcegraph`

See https://sourcegraph.sgdev.org/github.com/sourcegraph/sourcegraph/-/blob/cmd/frontend/internal/app/pkg/updatecheck/handler.go#L29:5

## Minikube

You can use minikube to run Sourcegraph Cluster on your development machine. However, due to minikube requirements and reduced available resources we need to modify the resources to remove `resources` requests/limits and `storageClassNames`. Here is the shell commands you can use to spin up minikube:

```shell
find base -name '*Deployment.yaml' | while read i; do yj < $i | jq 'walk(if type == "object" then del(.resources) else . end)' | jy -o $i; done
find base -name '*PersistentVolumeClaim.yaml' | while read i; do yj < $i | jq 'del(.spec.storageClassName)' | jy -o $i; done
find base -name '*StatefulSet.yaml' | while read i; do yj < $i | jq 'del(.spec.volumeClaimTemplates[] | .spec.storageClassName) | del(.spec.template.spec.containers[] | .resources)' | jy -o $i; done
minikube start
kubectl create ns src
kubens src
./kubectl-apply-all.sh
kubectl expose deployment sourcegraph-frontend --type=NodePort --name sourcegraph
kubectl expose deployment management-console --type=NodePort --name admin
minikube service list
```

Additionally you may want to deploy a modified version of a service locally. Minikube allows us to directly connect to its docker instance, making it easy to use unpublished images from the sourcegraph repository:

```shell
eval $(minikube docker-env)
IMAGE=repo-updater:dev ./cmd/repo-updater/build.sh
kubectl edit deployment/repo-updater # set imagePullPolicy to Never
kubectl set image deployment repo-updater '*=repo-updater:dev'
```
