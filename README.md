# Automating Cisco ACI with Ansible
This project is a starting skeleton for Ansible Automation Platform (AAP) against a Cisco ACI Simulator. By default it targets a homelab ACI Simulator VM, reachable over the public internet via port-forwarding. It can also be pointed at the [Cisco DevNet ACI Always-On sandbox](https://devnetsandbox.cisco.com/DevNet/catalog/ACI-Simulator-Always-On) instead — see the notes below. Two use cases are included:
1. Configuring Cisco ACI objects
2. Querying ACI Controller to collect ACI configuration

## Use this repo as an AAP project

Cloud-hosted AAP reaches the APIC over HTTPS — by default the homelab simulator's public IP (`80.138.119.248`), forwarded through the home router to the VM's out-of-band management interface; see the commented-out alternative in `inventory` to use the DevNet sandbox instead. This repo uses the **httpapi** connection plugin (`ansible.netcommon.httpapi` + `cisco.aci.aci`), so AAP must inject **`ansible_user` / `ansible_password`** (or the `ACI_USERNAME` / `ACI_PASSWORD` env vars that `group_vars/apic` maps onto those). Module-only env vars do nothing unless that mapping is present.

> **Security note:** Port-forwarding an APIC's OOB management interface (even a simulator) to the public internet is exposure you should limit. Prefer restricting the router's port-forward rule to AAP's known egress IP(s) if your router supports source-IP filtering, and never leave default/weak credentials on the exposed instance.

### 1. Create the project

In AAP: **Projects → Add**, source control type Git, URL of this repository. Sync it.

Collections are declared in `collections/requirements.yml`. The Organization needs an **Ansible Galaxy / Automation Hub** credential so AAP can install `cisco.aci` into the job environment. If that credential is missing, jobs fail with an error that `cisco.aci` cannot be found.

Use a default or network Execution Environment. The lab image in `ansible-navigator.yml` (`aap.rh.lab/...`) is not used by cloud AAP.

### 2. Inventory from the project

**Inventories → Add**, then add a source **Sourced from a Project**, file `inventory`.

That inventory points at the homelab APIC simulator's public IP by default. To use the DevNet Always-On sandbox (or a reserved, time-boxed simulator) instead, either uncomment its line in `inventory` or override `ansible_host` with extra vars.

### 3. Credential (required for login)

Do **not** put the APIC password in git. Use the admin credentials you set for your homelab simulator during its setup wizard (or, if targeting the DevNet sandbox, the current admin password from the DevNet sandbox page).

Pick one:

**Option A — Network credential (simplest)**  
Create a **Network** credential with the APIC username and password. Attach it to the Job Template. AAP injects `ansible_user` and `ansible_password`, which httpapi uses.

**Option B — Custom credential (matches this repo’s env vars)**  
Custom Credential **input**:

```
fields:
  - id: username
    type: string
    label: APIC Username
    secret: false
  - id: password
    type: string
    label: APIC Password
    secret: true
```

Custom Credential **injector**:

```
env:
  ACI_PASSWORD: '{{ password }}'
  ACI_USERNAME: '{{ username }}'
```

`group_vars/apic` copies those env vars into `ansible_user` / `ansible_password` for httpapi. Do not use a Machine/SSH credential for APIC.

### 4. Job Templates

Create templates against this project and the SCM inventory. No machine credential. Attach the Network or custom APIC credential. Suggested playbooks:

| Playbook | Purpose |
| --- | --- |
| `verify_connection.yml` | First run: login + tenant query |
| `configure_aci.yml` | Create sample tenant / VRF / BD / EPG / contract |
| `query_aci.yml` | Read tenant networking back from APIC |

Run `verify_connection.yml` before anything else. If that fails, the Job Template credential, EE collections, or outbound HTTPS to the APIC's IP/hostname on port 443 is wrong — not the playbook logic. For the homelab simulator, also confirm the router's port-forward and the VM are up.

### If using the DevNet sandbox instead

The Always-On APIC is a **shared** fabric. The sample data in `host_vars/apic1` uses tenant `production`. Change that name (and the `production` filter in `roles/query_apic`) per team or you will clash with other users. The sandbox is also rate-limited; httpapi is used so Ansible logs in once per play, not once per task. This does not apply to the homelab simulator, which is private and single-tenant.

### Local run (optional)

```bash
export ACI_USERNAME=admin
export ACI_PASSWORD='<your APIC password>'
ansible-playbook verify_connection.yml
```

`cisco.aci` must be installed locally (`ansible-galaxy collection install -r collections/requirements.yml`) or via an execution environment.

## Configuring Cisco ACI objects
Cisco ACI provides centralized approach to manage physical and virtual networks and one of its goals is to simplify and unify network management. But increased operational efficiency might not be fully experienced if we use manual operations via GUI to configure ACI. This so called 'ClickOps' requires many steps in GUI and is difficult to document (often requires screenshots of GUI configuration). Ansible provides powerful and robust approach to get rid of ClickOps and fully switch to Automation. In this example the following ACI Objects are configured with Ansible:
![image](https://github.com/mzdyb/cisco-aci/assets/49950423/c5446d6b-5c04-48a6-b44b-25317cc7ed70)


**Essential steps to automate with Ansible:**
1. Create inventory  
   In inventory we are defining the list of target hosts we want to automate. In our case it is ACI Controller
   ```
   [apic]
   apic1 ansible_host=80.138.119.248
   ```
   (This is the public IP of the homelab ACI Simulator, forwarded to its OOB management interface. Swap in `sandboxapicdc.cisco.com` to target the DevNet Always-On sandbox instead.)
   We are also defining variables related to connectivity to target host
   ```
   [apic:vars]
   ansible_connection=ansible.netcommon.httpapi
   ansible_network_os=cisco.aci.aci
   ansible_httpapi_use_ssl=true
   ansible_httpapi_validate_certs=false
   ansible_httpapi_port=443
   ```
   Username and password are supplied by AAP credentials (see above), not stored in this file.
   As we can see _httpapi plugin_ is used here to connect to ACI Controller. This is the recommended way to connect to APIC and offers a number of benefits like ability to defining credentials only once in inventory as opposed to including them in each task, authenticating per playbook run instead of per every tasks run etc. A good explanation of how _httpi plugin_ works and its benefits is provided here: [Cisco ACI httpapi plugin](https://www.ciscolive.com/on-demand/on-demand-library.html?search=httpapi#/session/1707505590105001pxJm)
     

3. Create data structure with variables reflecting ACI configuration  
   This is crucial step and in more advanced scenarios it might serve as Source of Truth (SoT) for our Cisco ACI configuration. SoT is central repository for configurations with up to date and reliable information. It serves as the reference point for desired state of the infrastructure. SoT documents our configuration so we don't need screenshots anymore and we are enabling full automation using Ansible. If we move SoT to Version Control System like GitHub we are also able to implement NetDevOps approach which brings benefits from Software Developement realm to network operations. The example of such approach is presented in the following GitHub project: [NetDevOps with Ansible Automation Platform](https://github.com/mzdyb/netdevops)

4. Write Ansible playbooks and use powerful Ansible Automation Platform features like Automation Workflows, RBAC etc.

   **Playbooks**  
   In this project Ansible Roles are used which allows to write modular/reusable automation code. Thanks to using Roles the whole playbook to implement our ACI Objects looks as simple as this:
   ```
   ---
   - name: Configure ACI
     hosts: apic

     roles:
       - configure_application
   ```
   **Automation Workflows**  
   Workflows allow creating visual logical representation of automation jobs sequence based on the result of the previous jobs run in the sequence. In this project modularity in Workflows is implemented by using Ansible Tags in Ansible Roles. The example of Automation Workflow from Ansible Automation Platform implementing automation defined in _configure_aci.yml_ playbook is shown below:

   ![image](https://github.com/mzdyb/cisco-aci/assets/49950423/22b57cfd-a4b0-447d-a153-27e3166cf091)
   The above approach is called _**Push of the button automation**_ as it allows to create sophisticated automation logic and to run it by just clicking "Run" button in the Workflow.

## Querying ACI Controller to collect ACI configuration
The second use case presented in this project is to collect ACI configuration from APIC. Ansible modules for ACI configuration have three possible states: 
- _present_ and _absent_ for adding and removing ACI Objects configuration
- _query_ for collecting Objects' configuration from Controller

In this example _query_ state is used to collect configuration of Tenant Networking objects: Tenants, VRFs, Bridge Domains and Subnets. This kind of query provides all configuration data related to queried objects and it might be a lot of data depending on queried objects so collected data is filtered to include only configuration parameters used by Ansible in the first use case in this project. For this purpose _json_query_ filter is used:
```
     {{ tenant_data.current |
         json_query("[?contains(fvTenant.attributes.dn,'production')].{
            name: fvTenant.attributes.name,
            description: fvTenant.attributes.descr
            }"
         )
      }}
```
As we can see only Tenant _name_ and _description_ are collected from APIC and only for Tenant named 'Production' which reflects configuration variables defined in _host_vars/apic1_ file.

## Remarks
Do not store the APIC password in inventory. For AAP, use a Network credential or the custom credential described in **Use this repo as an AAP project**. The custom credential injects `ACI_USERNAME` / `ACI_PASSWORD`; `group_vars/apic` maps those onto `ansible_user` / `ansible_password` so the httpapi plugin can log in.

If you are interested in ACI configuration approach with Source of True moved to Version Control System like GitHub/GitLab please check the ACI as a Code project from my Red Hat colleague Tony Dubiel: [ACI-as-Code](https://gitlab.com/redhatautomation/network_demos/-/blob/main/cisco_aci/)

## Feedback
Feedback is always welcome! If you have any comments, please reach me out

## Author

Original author: [@mzdyb](https://www.linkedin.com/in/michal-zdyb-9aa4046/)

