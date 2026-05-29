# AWX 

AWX provides a web-based user interface, REST API, and task engine built on top of Ansible. It is one of the upstream projects for Red Hat Ansible Automation Platform.

AWX will be installed as an Operator (awx-operator) on this sandbox using Kubernetes/Kustomize. This allows for each task to be self-contained in a Pod.

Note: As of this guide, AWX is currently being restructured and is still stuck on build `24.6.1` (awx-operator is `2.19.1`)from June 2024. More information on the links listed below

## Setup Instructions

1. Install AWX on Kubernetes using Kustomize

This step also creates 2 testing containers.
```sh
kubectl apply -k .
```

2. Wait for the operator to install

```sh
kubectl get pods -n awx
# It should look like this:
# NAME                                               READY   STATUS    RESTARTS   AGE
# awx-operator-controller-manager-66ccd8f997-rhd4z   2/2     Running   0          11s
```

3. Install the AWX Custom Resource 

This step is different, as it requires you to add the new AWX CR to the already build Kustomize file

```sh
cat <<EOF > ./manifests/awx-resource.yaml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  service_type: nodeport
EOF

echo "  - awx-resource.yaml" >> kustomization.yaml

kubectl apply -k .
```
4. Retrieve the credentials

```sh
# User: admin
kubectl get secret awx-admin-password -n ${NAMESPACE} -o jsonpath="{.data.password}" | base64 --decode
```

---
## Configuration

### Ansible playbooks & GitHub configuration

1. Create a new **private** repository on GitHub and configure a Deploy Key on it
2. Clone it to your local machine
```sh
git clone git@github.com:<YOUR_NAME>/<REPO_NAME>.git
```

3. Install the ansible toolkit on your machine
```sh
apt install ansible-core
```

4. Import the files in the folder gitrepo to your repository

### AWX configuration

1. Access AWX with the URL that Minikube provided
	- If it expired, use the following command:
```sh
minikube service awx-service -n awx --url &
```
	- If you didn't save the password, use the following command:
```sh
kubectl get secret awx-admin-password -n ${NAMESPACE} -o jsonpath="{.data.password}" | base64 --decode; echo
```

2. Add a new Credential for GitHub (*Resources > Credentials > Add*)
	- **Name**: GH-Token
	- **Organization**: Default
	- **Credential Type**: Source Control
	- **Username**: Your GitHub username
	- **SCM Private Key**: The private key of the Deploy Key defined on the repository

3. Add a new Project (*Resources > Projects > Add*)
	- **Name**: Private Ansible GH Repo
	- **Organization**: Default
	- **Source Control** Type: Git
	- **Source Control URL**: Your Git repo in SSH format (i.e. `git@github.com:<YOUR_NAME>/<REPO_NAME>.git`)
	- **Credentials**: GH-Token (as created before)
	- **Update Revision on Launch**: Checked

4. Create a new Inventory (*Resources > Inventories > Add > Add inventory*)
	- **Name**: Testing Machines

5. Create a new Source (*Resources > Inventories > Testing* Machines)
	- **Name**: GitHub Source
	- **Source**: Sourced from a Project
	- **Project**: Private Ansible GH Repo
	- **Inventory file**: /scm-inventory
	- **Overwrite**: Checked
	- **Update on launch**: Checked

 6. Create a new Credential for SSH (*Resources > Credentials > Add*)
	- **Name**: Testing Machines SSH
	- **Organization**: Default
	- **Credential Type**: Machine
	- **Username**: root
	- **SCM Private Key**: The private SSH key that was generated

7. Create two new templates (*Resources > Templates > Add > Add Job Template*)
	1. System Check
		- **Name**: System Check
		- **Inventory**:Testing Machines
		- **Project**: Private Ansible GH Repo
		- **Playbook**: system-check/main.yml
		- **Credentials**: Testing Machines SSH
	2. Nginx Install
		- **Name**: Nginx Install
		- **Inventory**: Testing Machines
		- **Project**: Private Ansible GH Repo
		- **Playbook**: nginx-install/main.yml
		- **Credentials**: Testing Machines SSH
	3. Set the templates' Variables:
```yml
ansible_connection: ssh
ansible_ssh_common_args: '-o StrictHostKeyChecking=no'
```

8. Add the following options to the surveys of the templates (Resources > Templates > TEMPLATE_NAME > Survey > Add):
	1. System Check
		1. **Question**: Machine
			- **Answer variable name**: machine
			- **Answer type**: Multiple Choice (multiple select)
			- **Required**: Checked
			- **Multiple Choice Options**: (Click on the check for all options)
				- ubuntu-service.awx.svc.cluster.local 
				- redhat-service.awx.svc.cluster.local
		2.  **Question**: Information Options
			- **Answer variable name**: info_opts
			- **Answer type**: Multiple Choice (multiple select)
			- **Required**: Checked
			- **Multiple Choice Options**: (Click on the check for all options)
				- OS
				- Kernel
				- IPv4
				- Memory
				- Processor
				- Volumes
				- Virtualization
				- Extras
		3. Save and click on **Survey Enabled**

---
## Testing

- Run the templates previously created and check the results
- If you wish to delete the OS images so you can rerun the playbooks for the first time, run the following command:
```sh
kubectl delete pod -n awx $(kubectl get pods -n awx | grep -E '^ubuntu|redhat' | cut -d' ' -f1)
```

---

## Cleanup

To uninstall AWX and all components:

```sh
kubectl delete namespace awx
```

## Useful Links

[Basic Install - Ansible AWX Operator Documentation](https://ansible.readthedocs.io/projects/awx-operator/en/latest/installation/basic-install.html)
[ansible/awx-operator: An Ansible AWX operator for Kubernetes built with Operator SDK and Ansible. 🤖](https://github.com/ansible/awx-operator)
[ansible/awx: AWX provides a web-based user interface, REST API, and task engine built on top of Ansible. It is one of the upstream projects for Red Hat Ansible Automation Platform.](https://github.com/ansible/awx)
[Streamlining AWX Releases - Project Discussions - Ansible](https://forum.ansible.com/t/streamlining-awx-releases/6894)
[Ansible Community | Ansible documentation](https://docs.ansible.com/)
