# Ansible Collection - irunasroot.servicenow
Collection to help manage various aspects of ServiceNow

# Features

## [Agent Client Collector](roles/agent_client_collector/README.md)
Installs and configures the Agent Client Collector (ACC) agent.

There's a base role that will install, configure, and restart the ACC on the inventory, but can also be broken down into
individual roles if needed. All the documenation reguarding this role can be found in the [role's readme](roles/agent_client_collector/README.md).

### Roles
* agent_client_collector
  * agent_client_collector/install
  * agent_client_collector/configuration
  * agent_client_collector/restart
  * agent_client_collector/uninstall
