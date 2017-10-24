# Linode-Dynamic-DNS
Dynamic script to update Linode's Managed DNS from a MikroTik router.


## Add the following script to the Scripts on the router.

```
# Linode automatic Dynamic DNS update

#--------------- Change Values in this section to match your setup ------------------

# Linode API Key 
:local linodeApiKey "your-api-key"

# Set the domain id(s) of the domain to be updated.
# This can be found at the following URL: 
# https://api.linode.com/?api_key=your-api-key&api_action=domain.list
# To specify multiple domains, separate them with commas.
:local linodeDomainIds "your-domain-ids"

# Set the resource id(s) of the domain to be updated.
# This can be found at the following URL: 
# https://api.linode.com/?api_key=your-api-key&&api_action=domain.resource.list&domainid=your-domain-id
# To specify multiple resource ids, separate them with commas. 
#   * If there is more than one domain id, these resource ids need to be the same length as the domain ids.
#   * Otherwise, it tries to set the resources for the one domain id.
:local linodeResourceIds "your-resource-ids"

# Change to the name of interface that gets the dynamic IP address
:local inetinterface "external-interface-name"

#------------------------------------------------------------------------------------
# No more changes need

:global linodePreviousIP

:if ([/interface get $inetinterface value-name=running]) do={
# Get the current IP on the interface
  :local linodeCurrentIP [/ip address get [find interface="$inetinterface" disabled=no] address]

# Strip the net mask off the IP address
  :for i from=( [:len $linodeCurrentIP] - 1) to=0 do={
     :if ( [:pick $linodeCurrentIP $i] = "/") do={ 
         :set linodeCurrentIP [:pick $linodeCurrentIP 0 $i]
     } 
  }

  :if ($linodeCurrentIP != $linodePreviousIP) do={
    :log info "Linode: Current IP $linodeCurrentIP is not equal to previous IP, update needed"
    :set linodePreviousIP $linodeCurrentIP

    # The update URL. Note the "\3F" is hex for question mark (?). Required since ? is a special character in commands.
    :local url "https://api.linode.com/\3Fapi_key=$linodeApiKey&api_action=domain.resource.update&target=$linodeCurrentIP"
    :local linodeDomainIdArray
    :local linodeResourceIdArray
    :set linodeDomainIdArray [:toarray $linodeDomainIds]
    :set linodeResourceIdArray [:toarray $linodeResourceIds]

    if ( [:len $linodeDomainIdArray] > 1 && [:len $linodeDomainIdArray] = [:len $linodeDomainIdArray]) do={
      :for i from=0 to=( [:len $linodeDomainIdArray] - 1) do={
        :local linodeDomainId [:pick $linodeDomainIdArray 0 $i]
        :local linodeResourceId [:pick $linodeResourceIdArray 0 $i]

        :log info "Linode: Sending update for domain id $linodeDomainId and resource id $linodeResourceId"
        /tool fetch url=($url . "&domainid=$linodeDomainId&resourceid=$linodeResourceId") mode=http 
        :log info "Linode: Domain id $linodeDomainId and resource id $linodeResourceId updated on Linode with IP $linodeCurrentIP"
      }

    } else={
      if ( [:len $linodeDomainIdArray] = 1) do={
        :local linodeDomainId $linodeDomainIds
        :foreach linodeResourceId in=$linodeResourceIdArray do={

          :log info "Linode: Sending update for domain id $linodeDomainId and resource id $linodeResourceId"
          /tool fetch url=($url . "&domainid=$linodeDomainId&resourceid=$linodeResourceId") mode=http 
          :log info "Linode: Domain id $linodeDomainId and resource id $linodeResourceId updated on Linode with IP $linodeCurrentIP"
        }
      } else={
        :log error "Linode: Configuration error cannot perform update: multiple domain ids must have a matching number of resource ids."
      }
    }
  } else={
    :log info "Linode: No update needed because IP $linodePreviousIP has not changed."
  }
} else={
  :log info "Linode: $inetinterface is not currently running, so therefore will not update."
}
```
## Then, Setup the Recurring Update

```
/system scheduler add comment="Update Linode DDNS" disabled=no interval=5m \
name="Update Linode DDNS" on-event=<script name> policy=read,write,test

```
