# NSX API enablement 

# Need govc 

```
sudo curl -L -o - "https://github.com/vmware/govmomi/releases/latest/download/govc_$(uname -s)_$(uname -m).tar.gz" > /tmp/a
sudo tar -C /usr/local/bin -xvzf /tmp/a govc

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
cat /tmp/a
#
# test
#
#Get the content lib ID (in my case entry number 1 (entry zero is something else))
govc Library.info 'Kubernetes Service Content Library'
```


# ENV variables needed for API call
```
export SUPERVISOR_NAME=""
export ZONE_ID=""
export StoragePolicy=""
export MgmtNetwork=""
export MGMT_GATEWAY_IP=""
export MGMT_PREFIX=""
export MGMT_STARTING_IP}=""
export DNS_SEARCHDOMAIN=""
export DNS_SERVER=""
export NTP_SERVER=""
export NSX_DVS=""
export NAMESPACE_CIDR=""
export NAMESPACE_COUNT=""
export SERVICE_CIDR=""
export SERVICE_COUNT=""
export DNS_SEARCHDOMAIN=""
export DNS_SERVER=""
export NTP_SERVER=""
export LB_ID=`govc library.info -json | jq -r '.[1].id'`
export INGRESS_CIDR=""
export INGRESS_COUNT=""
export NSX_EDGE_CLUSTER_ID=""
export NSX_T0_GATEWAY_ID=""
export EGRESS_CIDR=""
export EGRESS_COUNT=""
export K8S_CONTENT_LIBRARY_ID=""
export StoragePolicy=""
export wcp."${DNS_SEARCHDOMAIN}"=""
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



