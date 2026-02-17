Note & Limitations : 
- A private link service has a one-to-one relationship with a workspace. As shown in the diagram, each workspace has its own private link service.
- A workspace's private link service can have multiple private endpoints. For example, both VNet A and VNet B connect to Workspace 1 via separate private endpoints. The limit of the number of private endpoints can be found in Supported scenarios and limitations for workspace-level private links
- A virtual network can connect to multiple workspaces by creating separate private endpoints for each. For example, VNet B connects to Workspaces 1, 2, and 3 using three private endpoints.
- When connecting to a workspace, you need to use the workspace fully qualified domain name (FQDN). The workspace FQDN is constructed based on the workspace ID and the first two characters of the workspace object ID. The following are the formats for the workspace FQDN. The workspaceid is the workspace object ID without dashes, and xy represents the first two characters of the workspace object ID. Find the workspace object ID in the URL after group when opening the workspace page from Fabric portal. You can also get workspace FQDN by running List workspace API or Get workspace API.
```
https://{workspaceid}.z{xy}.w.api.fabric.microsoft.com
https://{workspaceid}.z{xy}.c.fabric.microsoft.com
https://{workspaceid}.z{xy}.onelake.fabric.microsoft.com
https://{workspaceid}.z{xy}.dfs.fabric.microsoft.com
https://{workspaceid}.z{xy}.blob.fabric.microsoft.com
```
- For data warehouse connection strings, use https://{GUID}-{GUID}.z{xy}.datawarehouse.fabric.microsoft.com that is, add z{xy} to the regular warehouse connection string found under SQL connection string. The GUIDs in the FQDN correspond to Tenant GUID in Base32 and Workspace GUID in Base32 respectively. This FQDN is not available as part of the DNS configurations for the private endp
- The workspace FQDN must be constructed correctly using the workspace object ID without dashes and the correct xy prefix (the first two characters of the workspace object ID). If the FQDN isn't formatted correctly, it doesn't resolve to the intended private IP address, and the workspace-level private link connection fails.
