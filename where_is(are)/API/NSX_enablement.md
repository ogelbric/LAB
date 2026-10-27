# NSX API enablement 

# Need govc 

```
sudo curl -L -o - "https://github.com/vmware/govmomi/releases/latest/download/govc_$(uname -s)_$(uname -m).tar.gz" > /tmp/a
sudo tar -C /usr/local/bin -xvzf /tmp/a govc

```
# ENV variables needed for API call
```
"${SUPERVISOR_NAME}",
"${ZONE_ID}",
"${StoragePolicy}",
"${MgmtNetwork}"
"${MGMT_GATEWAY_IP}/${MGMT_PREFIX}",
"${MGMT_STARTING_IP}",
"${DNS_SEARCHDOMAIN}"
"${DNS_SERVER}"
"${NTP_SERVER}"
"${NSX_DVS}"
"${NAMESPACE_CIDR}"
"${NAMESPACE_COUNT}
"${SERVICE_CIDR}"
${SERVICE_COUNT}
"${DNS_SEARCHDOMAIN}"
"${DNS_SERVER}"
"${NTP_SERVER}"
"${LB_ID}"
"${INGRESS_CIDR}"
"${INGRESS_COUNT}"
"${NSX_EDGE_CLUSTER_ID}"
"${NSX_T0_GATEWAY_ID}"
"${EGRESS_CIDR}"
"${EGRESS_COUNT}"
"${K8S_CONTENT_LIBRARY_ID}"
"${StoragePolicy}"
"${StoragePolicy}"
"wcp.${DNS_SEARCHDOMAIN}"
```
# Log onto VC to get some information

```
export GOVC_URL="vc-wld01-a.site-a.vcf.lab"
export GOVC_USERNAME="administrator@WLD.SSO"
export GOVC_PASSWORD='VMware123!VMware123!'
export GOVC_DATASTORE="vsan-wld01-01a"
export GOVC_INSECURE="true"in
export GOVC_DATACENTER="dc-a"
export GOVC_NETWORK="mgmt-vds01-wld01-01a"
#
#govc 2>&1 | tee /tmp/a
# to see all the govc commands
```
```
# ENV variables needed for API call
```
"${SUPERVISOR_NAME}"
"${ZONE_ID}",
"${StoragePolicy}"
"${MgmtNetwork}"
"${MGMT_GATEWAY_IP}/${MGMT_PREFIX}"
"${MGMT_STARTING_IP}"
"${DNS_SEARCHDOMAIN}"
"${DNS_SERVER}"
"${NTP_SERVER}"
"${NSX_DVS}"
"${NAMESPACE_CIDR}"
"${NAMESPACE_COUNT}
"${SERVICE_CIDR}"
${SERVICE_COUNT}
"${DNS_SEARCHDOMAIN}"
"${DNS_SERVER}"
"${NTP_SERVER}"
export LB_ID=`govc Library.info -json | jq -r '.[1].id'`
"${INGRESS_CIDR}"
"${INGRESS_COUNT}"
"${NSX_EDGE_CLUSTER_ID}"
"${NSX_T0_GATEWAY_ID}"
"${EGRESS_CIDR}"
"${EGRESS_COUNT}"
"${K8S_CONTENT_LIBRARY_ID}"
"${StoragePolicy}"
"wcp.${DNS_SEARCHDOMAIN}"
```

```
#Get the content lib ID
govc Library.info 'Kubernetes Service Content Library'

```

## Sample:
```
cat << EOF > /tmp/api.yaml
{
    "name": "${SUPERVISOR_NAME}",
    "zone": "${ZONE_ID}",
    "control_plane": {
        "count": 1,
        "size": "SMALL",
        "storage_policy": "${StoragePolicy}",
        "login_banner": "You are connected to Supervisor",
        "network": {
            "backing": {
                "backing": "NETWORK",
                "network": "${MgmtNetwork}"
            },
            "ip_management": {
                "dhcp_enabled": false,
                "gateway_address": "${MGMT_GATEWAY_IP}/${MGMT_PREFIX}",
                "ip_assignments": [ {
                    "assignee": "NODE",
                    "ranges": [ {
                        "address": "${MGMT_STARTING_IP}",
                        "count": 5
                    } ]
                } ]
            },
            "proxy": {
                "proxy_settings_source": "VC_INHERITED"
            },
            "services": {
                "dns": {
                    "search_domains": [ "${DNS_SEARCHDOMAIN}" ],
                    "servers": [ "${DNS_SERVER}" ]
                },
                "ntp": {
                    "servers": [ "${NTP_SERVER}" ]
                }
            }
        }
    },
    "workloads": {
        "network": {
            "network_type": "NSXT",
            "nsx": {
                "dvs": "${NSX_DVS}",
                "namespace_subnet_prefix": 28
            },
            "ip_management": {
                "dhcp_enabled": false,
                "ip_assignments": [
                    {
                        "assignee": "POD",
                        "ranges": [ {
                            "address": "${NAMESPACE_CIDR}",
                            "count": ${NAMESPACE_COUNT}
                        } ]
                    },
                    {
                        "assignee": "SERVICE",
                        "ranges": [ {
                            "address": "${SERVICE_CIDR}",
                            "count": ${SERVICE_COUNT}
                        } ]
                    }
                ]
            },
            "services": {
                "dns": {
                    "search_domains": [ "${DNS_SEARCHDOMAIN}" ],
                    "servers": [ "${DNS_SERVER}" ]
                },
                "ntp": {
                    "servers": [ "${NTP_SERVER}" ]
                }
            }
        },
        "edge": {
            "provider": "NSX",
            "id": "${LB_ID}",
            "load_balancer_address_ranges": [ {
                "address": "${INGRESS_CIDR}",
                "count": ${INGRESS_COUNT}
            } ],
            "nsx": {
                "edge_cluster_ID": "${NSX_EDGE_CLUSTER_ID}",
                "t0_gateway": "${NSX_T0_GATEWAY_ID}",
                "routing_mode": "NAT",
                "load_balancer_size": "SMALL",
                "egress_IP_ranges": [ {
                    "address": "${EGRESS_CIDR}",
                    "count": ${EGRESS_COUNT}
                } ]
            }
        },
        "images": {
            "kubernetes_content_library": "${K8S_CONTENT_LIBRARY_ID}"
        },
        "storage": {
            "ephemeral_storage_policy": "${StoragePolicy}",
            "image_storage_policy": "${StoragePolicy}"
        },
        "kube_API_server_options": {
            "security": {
                "certificate_dns_names": [ "wcp.${DNS_SEARCHDOMAIN}" ]
            }
        }
    }
}
EOF
```

# Need govc 

```
sudo curl -L -o - "https://github.com/vmware/govmomi/releases/latest/download/govc_$(uname -s)_$(uname -m).tar.gz" > /tmp/a
sudo tar -C /usr/local/bin -xvzf /tmp/a govc

```

# Need to get some ID's 

```
export GOVC_URL="vc-wld01-a.site-a.vcf.lab"
export GOVC_USERNAME="administrator@WLD.SSO"
export GOVC_PASSWORD='VMware123!VMware123!'
export GOVC_DATASTORE="vsan-wld01-01a"
export GOVC_INSECURE="true"in
export GOVC_DATACENTER="dc-a"
export GOVC_NETWORK="mgmt-vds01-wld01-01a"
#
govc 2>&1 | tee /tmp/a
# to see all the govc commands

#Get the content lib ID
govc Library.info 'Kubernetes Service Content Library'
```



