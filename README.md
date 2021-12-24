# backstage-demo

This demo is based on GitHub. It requires some manual preparation steps for tasks that do not seem automate-able on GitHub (at least i was no able to automate them).

## Manual preparation

1. create a new organization or reuse an existing one.
2. create an Oauth app in this organization for backstage. The call back url should be `https://backstage.apps.${based_domain}/api/auth/github`
3. create an Oauth app in this organization for Code Ready Workspaces. The call back url should be `https://codeready-openshift-workspaces.apps.${based_domain}/auth/realms/codeready/broker/github/endpoint`
4. create an Oauth app in this organization for OpenShift. The call back url should be `https://oauth-openshift.apps.${based_domain}/oauth2callback/backstage-demo-github/`
5. create a Personal Access Token (PAT) with an account that is administrator to the chosen organization.
6. create a GitHub application in this organization for the github action runner controller following the instructions [here](https://github.com/actions-runner-controller/actions-runner-controller#deploying-using-github-app-authentication). Store the ssh key pem in a file called `github_action_runner_app.pem`, it will be ignored by git. The callback url should be `https://ghr.apps.${based_domain}`. The webhook secret is hardcoded to `ciao`.
7. create a GitHub Application for the group-sync-operator following the instructions [here](https://github.com/redhat-cop/group-sync-operator#as-a-github-app). Store the ssh key pem in a file called `group-sync-operator-app-key.pem`, it will be ignored by git.

Create a client secret for each of the OAuth apps.

Create a file called `secrets.sh` and store it at the top of this repo, it will be ignored by Git.

```shell
export github_organization=<org_name>
export backstage_github_client_id=<backstage_oauth_app_id>
export backstage_github_client_secret=<backstage_oauth_app_secret>
export crw_github_client_id=<crw_oauth_app_id>
export crw_github_client_secret=<crw_oauth_app_secret>
export ocp_github_client_id=<ocp_oauth_app_id>
export ocp_github_client_secret=<ocp_oauth_app_secret>
export org_admin_pat=<pat token>
export action_runner_github_app_id=<application_id_for_action_runner>
export action_runner_github_app_installation_id=<application_installation_id_for_action_runner>
export action_runner_github_app_private_key_file_path=./github_action_runner_app.pem
export group_sync_github_app_id=<application_id_for_group_sync-operator>
export group_sync_operator_github_app_key_file_path=./group-sync-operator-app-key.pem
```

now you can source the file and populate the environment variables any time:

```shell
source ./secrets.sh
```

Run the following commands to populate the Kubernetes secrets with the previously generated values (this is fine for a demo, it might not be fine for a production environment):

```shell
oc new-project openshift-workspaces
oc create secret generic github-oauth-config --from-literal=id=${crw_github_client_id} --from-literal=secret=${crw_github_client_secret} -n openshift-workspaces
oc label secret github-oauth-config -n openshift-workspaces --overwrite=true app.kubernetes.io/part-of=che.eclipse.org app.kubernetes.io/component=oauth-scm-configuration
oc annotate secret github-oauth-config -n openshift-workspaces --overwrite=true che.eclipse.org/oauth-scm-server=github
oc create secret generic ocp-github-app-credentials -n openshift-config --from-literal=client_id=${ocp_github_client_id} --from-literal=clientSecret=${ocp_github_client_secret}
oc new-project backstage
oc create secret generic github-credentials -n backstage --from-literal=AUTH_GITHUB_CLIENT_ID=${backstage_github_client_id} --from-literal=AUTH_GITHUB_CLIENT_SECRET=${backstage_github_client_secret} --from-literal=GITHUB_TOKEN=${org_admin_pat} --from-literal=GITHUB_ORG=${github_organization}
oc new-project actions-runner-system
oc create secret generic controller-manager -n actions-runner-system --from-literal=github_app_id=${action-runner-github_app_id} --from-literal=github_app_installation_id=${action-runner-github_app_installation_id} --from-file=github_app_private_key=${action-runner-github_app_private_key_file_path}
oc new-project group-sync-operator
oc create secret generic github-group-sync -n group-sync-operator --from-literal=appId=${group_sync_github_app_id} --from-file=privateKey=${group_sync_operator_github_app_key_file_path}
```

To improve the demo experience and have some data pre-populated, you can optionally fork these repos to the new organization:

- `https://github.com/raf-backstage-demo/backstage`
- `https://github.com/raf-backstage-demo/software-templates`

The rest of the demo should be deployed by the gitops operator following the steps below.

## Deploy the gitops operator

```shell
oc apply -f ./argocd/operator.yaml
oc apply -f ./argocd/rbac.yaml
oc apply -f ./argocd/argocd.yaml
oc apply -f ./argocd/argo-root-application.yaml
```

This should be all to setup the demo.

Start enjoying the demo from here `https://backstage.apps.${based_domain}`.

## Notes

at the moment is still unclear what creates namespaces. regardless of that, namespace annotation are considered trusted and several security features revolve around them. These are the well known annotations:

- `app` : name of the app deployed to this namespace. This is used by the github runner to pick jobs from any component related to this app. For this to be secure, this piece of information needs to be trusted on the github workflow definition side.
- `team`: name of the team who owns this namespace (it will be considered an OCP group and given edit rights).
- `build-namespace`: the name of the namespace in which the pipeline which deploys to this namespace runs. In the build-namespace a github runner is deployed with edit permissions.
- `size`: determines the quota given to the namespace. Allowed values `small`, `medium`, `large`.
- `environment`: the purpose of the namespace. If environment equals build, an action runner for that app will be deployed. Other special behaviors related to environment might added in the future.
