## PostgreSQL HA Deployment

This postgres HA is configure using postgres operator (pgo), created by crunchy data teams. We will installe it via helm, follow the instruction provided here https://access.crunchydata.com/documentation/postgres-operator/latest/installation/helm. 
### Prerequisites
You need to fork this repository https://github.com/CrunchyData/postgres-operator-examples/fork
### Step By Step Installation
1. Clone your fork repository, for its https://github.com/purwaren/postgres-operator
```
git clone <YOUR REPO URL> postgres-operator
```
2. Go to the cloned repository, adjust the `helm/install/values.yml`
```
cd postgres-operator
```
If you want HA pgo, set the replica more than 1
3. Run the installation
```
kubectl create namespace postgres
helm install crunchy-postgres -n postgres helm/install
```
