# agent_client_collector

This role installs, configures, and restarts ServiceNow ACC agents. It can also uninstall if required.

Linux and Windows are supported. While ACC does support macOS the role is untested on macOS and doesn't include tasks for it.

This role only supports ACC Version 2.7.0 and above. This is due to how older versions used a different method for storing the agent's ID and this role is not setup to configure that method.

## Gotchas

### Windows
Installing packages via Ansible on Windows servers requires specifying the UUID of the MSI package so Ansible doesn't try to install the MSI every time it executes. This is found in the registry which is a pain to track down and requires the package to already be installed. There are various powershell or vbs scripts out there that could help, none of I will specify here so use at your own risk. If you have a known ProductCode of an ACC MSI package then feel free to update the list below.

| version | revision guid                          |
|---------|----------------------------------------|
| 2.7.0   | {B197E891-E6EB-40B9-94C4-AB0C802503F9} |
| 2.8.2   | {FA1D07AC-4DED-4A93-A32C-34184997386F} |
| 3.0.0   | {3179BAED-D98B-43D9-AB06-9C256372F564} |

If you are upgrading then providing the new MSI will upgrade the agent. If you're downgrading, you'll need to uninstall it first using the same `agent_client_collector.revision` attribute, but providing the old version and running `agent_client_collector/uninstall` directly.

### Variable merging
Due to the complex nature of the config files I've opted to create slightly more complex dictionary variables for `acc.yml` and `check-allow-list.json` files. And due to the 'complex' nature of variable merging, by default, Ansible disables this (most useful) feature. So if you need to add one or two things to something like `acc_check_allow_list` you'll need to either define the entire list or turn on [variable merging](https://docs.ansible.com/ansible/latest/reference_appendices/config.html#default-hash-behaviour). Its possible this (most useful) feature is disabled in the future, so use at your discretion.

### Agent ID & API Key
ACC agent will generate its own agent ID for encrypting the API key. The API key is then rewritten in the `acc.yml` config file as an encrypted hash. All this hinders the ability to have consistent configuration management as every time the playbook is executed it will overwrite the key with the unencrypted version, restart the agent, which will then encrypt it again.

To get around this you can specify the same agent-key-id and pre-encrypted api-key across all your agents. The only way to do this initially is to install the agent somewhere with the API key, then collect the two variables; I suggest putting this information in a vault of some kind.

For convince and without the need to define the entire `acc_configuration` variable when merging is turned off you can define these two variables with `acc_configuration_api_key` and `acc_configuration_agent_key_id` which `acc_configuration.api_key` and `acc_configuration.agent_key_id` will use respectively.

## Requirements
If using Windows you will need the required packages and tools for that specific login method as [defined by Ansible](https://docs.ansible.com/ansible/latest/user_guide/windows_winrm.html#authentication-options).

## Role Variables

| Variable                                                           | Role                   | Default                                                        | OS Type | Description                                                                                                                                                                                                             |
|--------------------------------------------------------------------|------------------------|----------------------------------------------------------------|---------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| force_uninstall_on_upgrade                                         | agent_client_collector | true                                                           | Linux   | Uninstall the incorrect version of the agent before doing an install. (There are lots of issues with post and pre RPM scripts, hence this option)                                                                       |
| restart_agent_after_config_changes                                 | agent_client_collector | true                                                           | Linux   | Restart the agent if config files have changed                                                                                                                                                                          |
| force_uninstall_on_error                                           | install                | true                                                           | Windows | The Windows MSI installer does funky things if it fails to install for some reason. In this case we just back out the install                                                                                           |
| linux_package_name                                                 | install                | agent-client-collector                                         | Linux   | The name of the Linux package to install                                                                                                                                                                                |
| linux_full_package_name                                            | install                | {{ linux_package_name }}-{{ agent_client_collector.version }}  | Linux   | The name of the Linux package with version                                                                                                                                                                              |
| agent_client_collector                                             | install                | Undefined                                                      | All     | The base key for defining ACC installation information                                                                                                                                                                  |
| agent_client_collector.version                                     | install                | Undefined                                                      | Linux   | The version of the agent to install. If `agent_client_collector.download_uri` is specified, then version is ignored.                                                                                                    |
| agent_client_collector.revision                                    | install                | Undefined                                                      | Windows | The MSI package UUID. This is needed so the MSI installer is not executed every time. See Requirements for more information this.                                                                                       |
| agent_client_collector.download_uri                                | Install                | Undefined                                                      | All     | The location of the install package. Either URL or a UNC path.                                                                                                                                                          |
| acc_yaml_file_path                                                 | configuration          | OS dependant                                                   | All     | The path to the `acc.yml` file. The role tries to resolve the default location unless you overwrite it                                                                                                                  |
| check_allow_list_json_file_path                                    | configuration          | OS dependant                                                   | All     | The path to the `check-allow-list.json` file. The role tries to resolve the default locations unless you overwrite it                                                                                                   |
| acc_configuration_api_key                                          | configuration          | Empty string                                                   | All     | The `acc.yml` api-key                                                                                                                                                                                                   |
| acc_configuration_agent_key_id                                     | configuration          | Empty string                                                   | All     | The `acc.yml` agent-key-id                                                                                                                                                                                              |
| acc_configuration                                                  | configuration          | [list too large to show here](configuration/defaults/main.yml) | All     | The base key for the `acc.yml` configuration file                                                                                                                                                                       |
| acc_configuration.name:                                            | configuration          | {{ ansible_fqdn }}                                             | All     | The name of the agent. Defaults to FQDN of the host this role is executing on                                                                                                                                           |
| acc_configuration.backend_url                                      | configuration          | Empty list                                                     | All     | A list of backend URLs in the format of `ws[s]://<url>:port/ws/events`                                                                                                                                                  |
| acc_configuration.api_key                                          | configuration          | {{ acc_configuration_api_key }}                                | All     | The API key. For mass deployment its suggested to include the already encrypted key with matching agent-key-id. If the unencrypted version is provided we rely on the agent to encrypt it.                              |
| acc_configuration.agent_key_id                                     | configuration          | {{ acc_configuration_agent_key_id }}                           | All     | The agent-key-id of the agent. For mass deployment its suggested to include the encrypted API key along with a pre-generated API key. If missing we dont insert it into the config and rely on the agent to generate it |
| acc_configuration.log_level                                        | configuration          | info                                                           | All     | The logging level of the agent                                                                                                                                                                                          |
| acc_configuration.skip_tls_verify                                  | configuration          | true                                                           | All     | Skip TLS verification on the MID server's SSL cert                                                                                                                                                                      |
| acc_configuration.allow_list                                       | configuration          | {{ check_allow_list_json_file_path }}                          | All     | The path to the `check-allow-list.json` file                                                                                                                                                                            |
| acc_configuration.redact                                           | configuration          | [list too large to show here](configuration/defaults/main.yml) | All     | A list of strings of variable names collected to redact before sending to ServiceNow                                                                                                                                    |
| acc_configuration.verify_plugin_signature                          | configuration          | true                                                           | All     | Verify plugin signatures                                                                                                                                                                                                |
| acc_configuration.max_running_checks                               | configuration          | 10                                                             | All     | The max number of checks that can be executed in parallel                                                                                                                                                               |
| acc_configuration.agent_cpu_threshold                              | configuration          |                                                                | All     | Defines CPU usage thresholds for the agent                                                                                                                                                                              |
| acc_configuration.agent_cpu_threshold.cpu_percentage_limit         | configuration          | 5                                                              | All     | CPU percentage limit                                                                                                                                                                                                    |
| acc_configuration.agent_cpu_threshold.repeated_high_cpu_num        | configuration          | 3                                                              | All     | Number of consecutive times the CPU percentage limit must be exceeded                                                                                                                                                   |
| acc_configuration.agent_cpu_threshold.monitor_interval_sec         | configuration          | 60                                                             | All     | Indicates that the monitor will run every X seconds.                                                                                                                                                                    |
| acc_configuration.agent_cpu_threshold.agent_cpu_threshold_disabled | configuration          | false                                                          | All     | Indicates whether gathering CPU threshold values for the agent protection feature is disabled                                                                                                                           |
| acc_configuration.agent_cpu_threshold.proxy_cpu_percentage_limit   | configuration          | 15                                                             | All     | CPU percentage limit when agent is running proxy check(s)                                                                                                                                                               |
| acc_configuration.disable_sockets: "true"                          | configuration          | true                                                           | All     |                                                                                                                                                                                                                         |
| acc_configuration.statsd_disable: "true"                           | configuration          | true                                                           | All     |                                                                                                                                                                                                                         |
| acc_configuration.check_commands_prefer_installed: "false"         | configuration          | false                                                          | All     | Specifies whether executables in the system PATH or ACC plugin executables have preference.                                                                                                                             |
| acc_configuration.enable_auto_mid_selection: "true"                | configuration          | true                                                           | All     | Enables or disables the Auto MID Selection feature. Disabling this feature means manual input of MID servers URLs using `acc_configuration.backend_url` attribute                                                       |
| acc_check_allow_list                                               | configuration          | [list too large to show here](configuration/defaults/main.yml) | All     | Base key to define the contents of `check-allow-list.json`                                                                                                                                                              |

The acc_check_allow_list attribute is a huge list of script exceptions and their allowable attributes. The format of each item in the list is as follows:

| Variable       | Description                                                                                                                                                                                                                                         |
|----------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| exec           | The name of the script that is going to be executed                                                                                                                                                                                                 |
| skip_arguments | Whether or not the arguments should be checked or not. True means the arguments have to be validated before execution                                                                                                                               |
| args           | A list of allow args that can be passed into the script. By default this list will be blank if not defined. If you want to define it but dont have requirements be sure to create this as an empty list otherwise the agent fails to read this file |

Example:
```yaml
acc_check_allow_list:
  - exec: "check-cpu.rb"
    skip_arguments: "true"
    args: []
  - exec: "cscript"
    args:
      - "check_ad.vbs //nologo"
    skip_arguments: "false"
```

## Example Playbook
```yaml
- name: Install and configure ACC agent
  collections:
    - irunasroot.servicenow
  hosts: servers
  vars:
    agent_client_collector:
      version: "2.8.2"
  roles:
    - name: "agent_client_collector"
```

```shell
ansible-playbook -i my_servers.yaml deploy.yaml --become --ask-pass --ask-become-pass
```

You can also pick and choose specific roles, for example, if you only want to configure and restart your agents, you can just simply include the `agent_client_collector/configuration` and `agent_client_collector/restart`
```yaml
- name: Configure and restart ACC agent
  collections:
    - irunasroot.servicenow
  hosts: servers
  roles:
    - name: "agent_client_collector/configuration"
    - name: "agent_client_collector/restart"
```

### Windows
Connecting to Windows hosts uses WinRM under the hood. For authentication, you can use [various methods](https://docs.ansible.com/ansible/latest/user_guide/windows_winrm.html#authentication-options) but my suggestion for best results is to use [psrp](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/psrp_connection.html).

Inventory vars for Windows hosts (psrp):
```yaml
ansible_become_method: runas
ansible_connection: psrp
ansible_psrp_auth: kerberos
ansible_psrp_cert_validation: ignore
```

Your playbook might look something like:
```yaml
- name: Install and configure ACC agent on Windows
  collections:
    - irunasroot.servicenow
  hosts: servers
  vars:
    agent_client_collector:
      version: "2.8.2"
      revision: "{FA1D07AC-4DED-4A93-A32C-34184997386F}"
      download_uri: "\\\\mydomain.local\\SYSVOL\\mydomain.local\\packages\\agent-client-collector-2.8.2-windows-x64.msi"
    ansible_become_method: runas
    ansible_connection: psrp
    ansible_psrp_auth: kerberos
    ansible_psrp_cert_validation: ignore
  roles:
    - name: "agent_client_collector"
```

```shell
ansible-playbook -i my_servers.yaml deploy.yaml --user me@MYDOMAIN.LOCAL --become-user admin.user@MYDOMAIN.LOCAL --become --ask-pass --ask-become-pass
```

## Testing
Using molecule to test the roles is included, however its only really working on EL7 due to ServiceNow's lack of testing their RPM packages, they are not signed properly, so direct installation from their install URL doesn't work on later versions of EL.

## License

GPL-3.0 or later
