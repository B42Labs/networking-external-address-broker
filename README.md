# networking-external-address-broker

Design and implementation document for a Neutron IPAM plugin and an associated
broker service that centrally manages floating IP addresses on external networks
across multiple OpenStack CloudPods.

## 1. Problem Statement

In a multi-CloudPod environment, multiple independent OpenStack control planes
share the same public IP network (e.g. 198.51.100.0/24). Each CloudPod has its
own external network with identical subnets. Without central coordination, the
same floating IP can be assigned in multiple CloudPods simultaneously.

Each assigned floating IP is announced by the respective CloudPod as a /32 route
via BGP. Duplicate assignments lead to routing conflicts.

## 2. Target Architecture

```
                        +-----------------------+
                        |   EAB Master          |
                        |   (central DB)        |
                        |                       |
                        |  Source of truth      |
                        |  for all IP           |
                        |  allocations          |
                        +--+--------+--------+--+
                           |        |        |
              +------------+        |        +--------------+
              |                     |                       |
     +--------v----------+  +-------v-----------+  +--------v----------+
     |  EAB Agent        |  |  EAB Agent        |  |  EAB Agent        |
     |  CloudPod A       |  |  CloudPod B       |  |  CloudPod C       |
     |                   |  |                   |  |                   |
     |  local cache      |  |  local cache      |  |  local cache      |
     +--------^----------+  +-------^-----------+  +--------^----------+
              |                     |                       |
     +--------+----------+  +-------+-----------+  +--------+----------+
     |  Neutron          |  |  Neutron          |  |  Neutron          |
     |  IPAM Driver      |  |  IPAM Driver      |  |  IPAM Driver      |
     |  (plugin)         |  |  (plugin)         |  |  (plugin)         |
     +-------------------+  +-------------------+  +-------------------+
```

Components:

- **EAB Master**: Central service. Maintains the authoritative database of all
  IP allocations. Runs once for the entire environment.
- **EAB Agent**: Local service per CloudPod. Communicates with the master and
  provides a local API for the Neutron IPAM driver. Maintains a local cache
  of the allocation state.
- **IPAM Driver**: Neutron plugin (out-of-tree). For floating IP allocations
  on external networks, the local EAB agent is contacted. All other
  allocations are delegated to the standard IPAM driver (NeutronDbPool).

## 3. Network-Side Counterpart: ovn-route-agent

The External Address Broker ensures that floating IPs are uniquely allocated
across all CloudPods. However, once an IP is allocated, it still needs to be
reachable from outside — this is where the
[ovn-route-agent](https://github.com/osism/ovn-route-agent) comes in.

The `ovn-route-agent` is an event-driven daemon that runs on the **Network Nodes**
(gateway chassis) of each CloudPod. It watches the OVN Southbound and Northbound
databases in real time via the OVSDB protocol and synchronizes the routing state
for floating IPs into the local network stack.

### 3.1 What it does

When a floating IP is assigned (via Neutron / the EAB), the ovn-route-agent
detects the corresponding NAT entry in OVN and:

1. **Creates a /32 host route** in the kernel for the floating IP
2. **Announces the /32 route via BGP** through FRR, so that upstream routers
   know which Network Node handles traffic for that specific IP
3. **Installs OVS flows** on the provider bridge (typically `br-ex`) to ensure
   proper packet handling (MAC-tweak flows for correct L2 acceptance)
4. **Manages VRF route leaking** via automatically-created veth pairs to connect
   the default and provider VRFs, enabling policy-based routing for return traffic

When a floating IP is removed, the agent cleans up all of these resources.

### 3.2 Why this matters for the EAB

In a multi-CloudPod setup, the combination of both components ensures
conflict-free routing:

```
                         ┌───────────────────────┐
                         │   Upstream Router     │
                         │   (receives BGP)      │
                         └───────┬──────┬────────┘
                                 │      │
               /32 route to A ───┘      └─── /32 route to B
                                 │      │
              ┌──────────────────┘      └──────────────────┐
              │                                            │
   ┌──────────▼───────────┐                 ┌──────────────▼──────────┐
   │  Network Node        │                 │  Network Node           │
   │  CloudPod A          │                 │  CloudPod B             │
   │                      │                 │                         │
   │  ovn-route-agent     │                 │  ovn-route-agent        │
   │  announces:          │                 │  announces:             │
   │    198.51.100.42/32  │                 │    198.51.100.43/32     │
   │    198.51.100.50/32  │                 │    198.51.100.51/32     │
   └──────────────────────┘                 └─────────────────────────┘
```

- The **EAB** guarantees that `.42` is only ever allocated in CloudPod A and
  `.43` only in CloudPod B — no duplicates across CloudPods.
- The **ovn-route-agent** ensures that only the Network Node holding the
  respective floating IP announces its /32 route via BGP — no conflicting
  routes from different chassis.

Without the ovn-route-agent, the /32 routes would not be announced and external
traffic would have no path to reach the floating IPs. Without the EAB, two
CloudPods could allocate the same IP and both announce conflicting /32 routes.

### 3.3 Prerequisites

The ovn-route-agent requires:

- TCP access to the OVN Southbound and Northbound databases
- FRR with `vtysh` available for BGP route injection
- An existing provider bridge (typically `br-ex`)
- Root or `CAP_NET_ADMIN` capabilities

For installation and configuration details, see the
[ovn-route-agent repository](https://github.com/osism/ovn-route-agent).

## 4. Component: Neutron IPAM Driver

### 4.1 Repository Structure

```
networking-external-address-broker/
├── setup.cfg
├── setup.py
├── pyproject.toml
├── README.rst
├── requirements.txt
├── test-requirements.txt
├── tox.ini
├── networking_external_address_broker/
│   ├── __init__.py
│   ├── version.py
│   ├── ipam/
│   │   ├── __init__.py
│   │   └── driver.py
│   ├── client/
│   │   ├── __init__.py
│   │   └── agent_client.py
│   ├── callbacks/
│   │   ├── __init__.py
│   │   └── floating_ip.py
│   ├── common/
│   │   ├── __init__.py
│   │   └── config.py
│   └── tests/
│       ├── __init__.py
│       ├── unit/
│       │   ├── __init__.py
│       │   ├── test_driver.py
│       │   └── test_callbacks.py
│       └── functional/
│           ├── __init__.py
│           └── test_integration.py
├── eab_agent/
│   ├── __init__.py
│   ├── app.py
│   ├── cache.py
│   ├── config.py
│   ├── master_client.py
│   └── tests/
│       └── ...
├── eab_master/
│   ├── __init__.py
│   ├── app.py
│   ├── db/
│   │   ├── __init__.py
│   │   ├── models.py
│   │   └── migration/
│   │       └── alembic/
│   │           └── versions/
│   ├── api/
│   │   ├── __init__.py
│   │   └── v1.py
│   ├── config.py
│   └── tests/
│       └── ...
└── etc/
    ├── eab-agent.conf.sample
    └── eab-master.conf.sample
```

### 4.2 setup.cfg

```ini
[metadata]
name = networking-external-address-broker
summary = Neutron IPAM driver for shared floating IPs across CloudPods
description_file = README.rst
license = Apache-2.0
classifier =
    Environment :: OpenStack
    Intended Audience :: Information Technology
    License :: OSI Approved :: Apache Software License
    Operating System :: POSIX :: Linux
    Programming Language :: Python :: 3
    Programming Language :: Python :: 3.10
    Programming Language :: Python :: 3.11

[options]
packages = find:
python_requires = >=3.10
install_requires =
    neutron-lib>=3.0.0
    oslo.config>=9.0.0
    oslo.log>=5.0.0
    requests>=2.28.0
    fastapi>=0.115.0
    uvicorn>=0.34.0

[options.packages.find]
exclude =
    *.tests.*

[entry_points]
neutron.ipam_drivers =
    external_address_broker = networking_external_address_broker.ipam.driver:ExternalAddressBrokerPool

oslo.config.opts =
    networking_external_address_broker = networking_external_address_broker.common.config:list_opts

console_scripts =
    eab-agent = eab_agent.app:main
    eab-master = eab_master.app:main
```

### 4.3 Configuration (neutron.conf)

```ini
[DEFAULT]
ipam_driver = external_address_broker

[external_address_broker]
# URL of the local EAB agent
agent_endpoint = http://localhost:9532

# Timeout for requests to the agent (seconds)
agent_timeout = 5

# Whether the driver should fall back to local allocation on agent failure.
# WARNING: When set to true, duplicate allocations are possible.
fallback_to_local = false
```

### 4.4 Config Module

```python
# networking_external_address_broker/common/config.py

from oslo_config import cfg

EXTERNAL_ADDRESS_BROKER_GROUP = cfg.OptGroup(
    name='external_address_broker',
    title='External Address Broker Options',
)

EXTERNAL_ADDRESS_BROKER_OPTS = [
    cfg.URIOpt(
        'agent_endpoint',
        default='http://localhost:9532',
        help='URL of the local EAB agent.',
    ),
    cfg.IntOpt(
        'agent_timeout',
        default=5,
        min=1,
        max=30,
        help='Timeout for requests to the EAB agent in seconds.',
    ),
    cfg.BoolOpt(
        'fallback_to_local',
        default=False,
        help='Fall back to local allocation on agent failure. '
             'WARNING: May lead to duplicate allocations.',
    ),
]


def register_opts(conf):
    conf.register_group(EXTERNAL_ADDRESS_BROKER_GROUP)
    conf.register_opts(EXTERNAL_ADDRESS_BROKER_OPTS,
                       group=EXTERNAL_ADDRESS_BROKER_GROUP)


def list_opts():
    return [(EXTERNAL_ADDRESS_BROKER_GROUP, EXTERNAL_ADDRESS_BROKER_OPTS)]
```

### 4.5 Agent Client (IPAM Driver -> local agent)

```python
# networking_external_address_broker/client/agent_client.py

import requests as http_requests
from oslo_config import cfg
from oslo_log import log

from networking_external_address_broker.common import config as eab_config

LOG = log.getLogger(__name__)


class AgentClientError(Exception):
    pass


class AgentUnavailableError(AgentClientError):
    pass


class AddressUnavailableError(AgentClientError):
    pass


class AgentClient:
    """Client for communication with the local EAB agent."""

    def __init__(self):
        eab_config.register_opts(cfg.CONF)
        self._endpoint = cfg.CONF.external_address_broker.agent_endpoint
        self._timeout = cfg.CONF.external_address_broker.agent_timeout
        self._session = http_requests.Session()

    def reserve(self, network_id, subnet_id=None, project_id=None):
        """Reserves an IP address via the agent from the master.

        :param network_id: Neutron network ID of the external network
        :param subnet_id: Optional restriction to a specific subnet
        :param project_id: Project/tenant ID
        :returns: dict with 'ip_address' and 'subnet_id'
        :raises: AgentUnavailableError, AddressUnavailableError
        """
        payload = {
            'network_id': network_id,
            'project_id': project_id,
        }
        if subnet_id:
            payload['subnet_id'] = subnet_id

        try:
            resp = self._session.post(
                f'{self._endpoint}/v1/addresses/reserve',
                json=payload,
                timeout=self._timeout,
            )
        except http_requests.ConnectionError:
            raise AgentUnavailableError(
                f'EAB agent unreachable: {self._endpoint}'
            )
        except http_requests.Timeout:
            raise AgentUnavailableError(
                f'EAB agent timeout after {self._timeout}s'
            )

        if resp.status_code == 200:
            return resp.json()
        elif resp.status_code == 409:
            raise AddressUnavailableError(resp.json().get('detail', ''))
        elif resp.status_code == 503:
            raise AgentUnavailableError(resp.json().get('detail', ''))
        else:
            raise AgentClientError(
                f'Unexpected status {resp.status_code}: {resp.text}'
            )

    def reserve_specific(self, network_id, ip_address, project_id=None):
        """Reserves a specific IP address.

        :param network_id: Neutron network ID
        :param ip_address: The requested IP address
        :param project_id: Project/tenant ID
        :returns: dict with 'ip_address' and 'subnet_id'
        :raises: AgentUnavailableError, AddressUnavailableError
        """
        payload = {
            'network_id': network_id,
            'ip_address': ip_address,
            'project_id': project_id,
        }

        try:
            resp = self._session.post(
                f'{self._endpoint}/v1/addresses/reserve',
                json=payload,
                timeout=self._timeout,
            )
        except (http_requests.ConnectionError, http_requests.Timeout) as e:
            raise AgentUnavailableError(str(e))

        if resp.status_code == 200:
            return resp.json()
        elif resp.status_code == 409:
            raise AddressUnavailableError(
                f'IP {ip_address} is already allocated'
            )
        else:
            raise AgentClientError(
                f'Unexpected status {resp.status_code}: {resp.text}'
            )

    def release(self, ip_address, network_id):
        """Releases an IP address.

        :param ip_address: The IP address to release
        :param network_id: Neutron network ID
        """
        payload = {
            'ip_address': ip_address,
            'network_id': network_id,
        }

        try:
            resp = self._session.post(
                f'{self._endpoint}/v1/addresses/release',
                json=payload,
                timeout=self._timeout,
            )
        except (http_requests.ConnectionError, http_requests.Timeout):
            LOG.error(
                'EAB agent unreachable during release of %s. '
                'Address must be released manually.',
                ip_address,
            )
            return

        if resp.status_code not in (200, 404):
            LOG.error(
                'Error releasing %s: %s %s',
                ip_address, resp.status_code, resp.text,
            )
```

### 4.6 IPAM Driver

```python
# networking_external_address_broker/ipam/driver.py

from neutron_lib import constants
from neutron_lib.plugins import directory
from oslo_config import cfg
from oslo_log import log

from neutron.ipam import driver as ipam_base
from neutron.ipam.drivers.neutrondb_ipam import driver as neutrondb_driver
from neutron.ipam import requests as ipam_req

from networking_external_address_broker.client import agent_client
from networking_external_address_broker.common import config as eab_config

LOG = log.getLogger(__name__)


def _is_external_network(context, network_id):
    """Checks whether a network is an external network."""
    plugin = directory.get_plugin()
    network = plugin.get_network(context, network_id)
    return network.get('router:external', False)


class ExternalAddressBrokerAddressRequestFactory(
        ipam_req.AddressRequestFactory):
    """AddressRequestFactory that routes floating IP allocations on external
    networks through the EAB agent.

    For all other allocations, the default behavior is used.
    """

    @classmethod
    def get_request(cls, context, port, ip_dict):
        # Only intercept floating IP ports on external networks
        if (port.get('device_owner') == constants.DEVICE_OWNER_FLOATINGIP
                and not ip_dict.get('ip_address')):
            network_id = port.get('network_id')
            try:
                if _is_external_network(context, network_id):
                    return cls._request_from_broker(
                        context, port, ip_dict,
                    )
            except Exception:
                LOG.exception(
                    'Error checking external network status for %s',
                    network_id,
                )

        # Specific IP address requested for a floating IP
        if (port.get('device_owner') == constants.DEVICE_OWNER_FLOATINGIP
                and ip_dict.get('ip_address')):
            network_id = port.get('network_id')
            try:
                if _is_external_network(context, network_id):
                    return cls._request_specific_from_broker(
                        context, port, ip_dict,
                    )
            except Exception:
                LOG.exception(
                    'Error during specific broker reservation for %s',
                    ip_dict['ip_address'],
                )

        # Default for everything else (DHCP, router, regular ports, ...)
        return super().get_request(context, port, ip_dict)

    @classmethod
    def _request_from_broker(cls, context, port, ip_dict):
        """Automatic IP allocation via the broker."""
        client = agent_client.AgentClient()
        fallback = cfg.CONF.external_address_broker.fallback_to_local

        try:
            result = client.reserve(
                network_id=port['network_id'],
                subnet_id=ip_dict.get('subnet_id'),
                project_id=port.get('project_id') or port.get('tenant_id'),
            )
            LOG.info(
                'EAB: IP %s reserved for network %s',
                result['ip_address'], port['network_id'],
            )
            # Update subnet_id in ip_dict so the IPAM allocator
            # uses the correct subnet
            if result.get('subnet_id'):
                ip_dict['subnet_id'] = result['subnet_id']
            return ipam_req.SpecificAddressRequest(result['ip_address'])

        except agent_client.AgentUnavailableError:
            if fallback:
                LOG.warning(
                    'EAB agent unreachable, falling back to local '
                    'allocation for network %s',
                    port['network_id'],
                )
                return ipam_req.AnyAddressRequest()
            raise

        except agent_client.AddressUnavailableError:
            from neutron_lib import exceptions as n_exc
            raise n_exc.ExternalIpAddressExhausted(
                net_id=port['network_id'],
            )

    @classmethod
    def _request_specific_from_broker(cls, context, port, ip_dict):
        """Validate a specific IP request via the broker."""
        client = agent_client.AgentClient()

        try:
            result = client.reserve_specific(
                network_id=port['network_id'],
                ip_address=ip_dict['ip_address'],
                project_id=port.get('project_id') or port.get('tenant_id'),
            )
            LOG.info(
                'EAB: Specific IP %s reserved for network %s',
                result['ip_address'], port['network_id'],
            )
            return ipam_req.SpecificAddressRequest(result['ip_address'])

        except agent_client.AddressUnavailableError:
            from neutron.ipam import exceptions as ipam_exc
            raise ipam_exc.IpAddressAlreadyAllocated(
                subnet_id=ip_dict.get('subnet_id', 'unknown'),
                ip=ip_dict['ip_address'],
            )


class ExternalAddressBrokerPool(neutrondb_driver.NeutronDbPool):
    """IPAM pool that overrides the AddressRequestFactory.

    All subnet operations (allocate_subnet, update_subnet, remove_subnet,
    get_subnet) are handled unchanged by NeutronDbPool.
    Only address allocation for floating IPs is routed through the broker.
    """

    def get_address_request_factory(self):
        return ExternalAddressBrokerAddressRequestFactory
```

### 4.7 Floating IP Delete Callback

```python
# networking_external_address_broker/callbacks/floating_ip.py

from neutron_lib.callbacks import events
from neutron_lib.callbacks import registry
from neutron_lib.callbacks import resources
from oslo_log import log

from networking_external_address_broker.client import agent_client

LOG = log.getLogger(__name__)


@registry.receives(resources.FLOATING_IP, [events.AFTER_DELETE])
def release_floating_ip_on_delete(resource, event, trigger, payload):
    """Releases the floating IP at the broker when it is deleted
    in Neutron."""
    fip = payload.latest_state
    if not fip:
        LOG.warning('EAB: AFTER_DELETE event received without FIP data')
        return

    ip_address = fip.get('floating_ip_address')
    network_id = fip.get('floating_network_id')

    if not ip_address or not network_id:
        return

    LOG.info('EAB: Releasing IP %s on network %s', ip_address, network_id)
    client = agent_client.AgentClient()
    client.release(ip_address=ip_address, network_id=network_id)
```

### 4.8 Callback Registration

Registration happens automatically via the `@registry.receives` decorator.
For the callback to be loaded, the module must be imported. This is done
via an additional entry point:

```ini
# In setup.cfg
neutron.policies =
    eab_callbacks = networking_external_address_broker.callbacks.floating_ip
```

Alternatively, a `NeutronModule` entry point can be used, or the import
can be ensured in the package's `__init__.py`.

## 5. Component: EAB Agent (per CloudPod)

The EAB agent runs as a standalone service on each CloudPod. It mediates
between the Neutron IPAM driver (local) and the EAB master (central).

### 6.1 Responsibilities

- Accepts reserve/release requests from the IPAM driver
- Forwards these to the EAB master
- Maintains a local cache of current allocations
- Performs periodic sync reconciliation with the master
- Reports its health status to the master

### 5.2 Configuration (eab-agent.conf)

```ini
[DEFAULT]
log_file = /var/log/eab/eab-agent.log

[agent]
# Unique ID of this CloudPod
cloudpod_id = cloudpod-a

# Display name
cloudpod_name = CloudPod Frankfurt 1

# Bind address for the local API
bind_host = 127.0.0.1
bind_port = 9532

[master]
# URL of the EAB master
endpoint = https://eab-master.example.com:9533

# Authentication towards the master
auth_token = <shared-secret-or-jwt>

# Timeout for master requests (seconds)
timeout = 10

# Retry behavior on master failure
max_retries = 3
retry_interval = 1

[sync]
# Interval for periodic full sync (seconds)
full_sync_interval = 60

# Interval for heartbeat to master (seconds)
heartbeat_interval = 10

[cache]
# Local cache for allocations
# Allows fast lookups without master roundtrip
enabled = true
```

### 5.3 API (local, only reachable from Neutron)

#### POST /v1/addresses/reserve

Reserves an IP address. Forwards the request to the master.

Request:
```json
{
    "network_id": "uuid-of-external-network",
    "subnet_id": "uuid-of-subnet (optional)",
    "project_id": "uuid-of-project"
}
```

Response (200):
```json
{
    "ip_address": "198.51.100.42",
    "subnet_id": "uuid-of-subnet",
    "network_id": "uuid-of-external-network",
    "reservation_id": "uuid-of-reservation"
}
```

Errors:
- 409: No free address available / specific address already allocated
- 503: Master unreachable (and no fallback configured)

#### POST /v1/addresses/release

Releases an IP address.

Request:
```json
{
    "ip_address": "198.51.100.42",
    "network_id": "uuid-of-external-network"
}
```

Response (200):
```json
{
    "ip_address": "198.51.100.42",
    "status": "released"
}
```

#### GET /v1/health

Health check endpoint.

Response (200):
```json
{
    "status": "healthy",
    "master_connected": true,
    "last_sync": "2026-03-19T10:30:00Z",
    "cache_size": 42
}
```

### 5.4 Implementation

```python
# eab_agent/app.py

import asyncio
from contextlib import asynccontextmanager

import uvicorn
from fastapi import FastAPI, HTTPException
from oslo_config import cfg
from oslo_log import log
from pydantic import BaseModel

from eab_agent import cache as agent_cache
from eab_agent import config as agent_config
from eab_agent import master_client

LOG = log.getLogger(__name__)

_master = None
_cache = None
_config = None


class ReserveRequest(BaseModel):
    network_id: str
    subnet_id: str | None = None
    ip_address: str | None = None
    project_id: str | None = None


class ReleaseRequest(BaseModel):
    ip_address: str
    network_id: str


def setup():
    global _master, _cache, _config
    agent_config.register_opts(cfg.CONF)
    cfg.CONF(project='eab-agent')
    log.setup(cfg.CONF, 'eab-agent')

    _config = cfg.CONF
    _master = master_client.MasterClient(
        endpoint=_config.master.endpoint,
        auth_token=_config.master.auth_token,
        timeout=_config.master.timeout,
        cloudpod_id=_config.agent.cloudpod_id,
        max_retries=_config.master.max_retries,
        retry_interval=_config.master.retry_interval,
    )
    _cache = agent_cache.AllocationCache()


async def _sync_loop():
    """Periodic reconciliation with the master."""
    while True:
        try:
            allocations = _master.get_allocations()
            _cache.full_sync(allocations)
            LOG.debug('Full sync completed, %d entries', len(allocations))
        except Exception:
            LOG.exception('Error during full sync')
        await asyncio.sleep(_config.sync.full_sync_interval)


async def _heartbeat_loop():
    """Periodic heartbeat to the master."""
    while True:
        try:
            _master.heartbeat()
        except Exception:
            LOG.warning('Heartbeat to master failed')
        await asyncio.sleep(_config.sync.heartbeat_interval)


@asynccontextmanager
async def lifespan(application: FastAPI):
    """Start background tasks on startup, cancel on shutdown."""
    sync_task = asyncio.create_task(_sync_loop())
    heartbeat_task = asyncio.create_task(_heartbeat_loop())
    yield
    sync_task.cancel()
    heartbeat_task.cancel()


app = FastAPI(lifespan=lifespan)


@app.post('/v1/addresses/reserve')
def reserve_address(data: ReserveRequest):
    try:
        if data.ip_address:
            result = _master.reserve_specific(
                network_id=data.network_id,
                ip_address=data.ip_address,
                project_id=data.project_id,
            )
        else:
            result = _master.reserve(
                network_id=data.network_id,
                subnet_id=data.subnet_id,
                project_id=data.project_id,
            )
    except master_client.MasterUnavailableError as e:
        raise HTTPException(status_code=503, detail=str(e))
    except master_client.AddressConflictError as e:
        raise HTTPException(status_code=409, detail=str(e))

    _cache.add(result['ip_address'], result)
    return result


@app.post('/v1/addresses/release')
def release_address(data: ReleaseRequest):
    try:
        _master.release(
            ip_address=data.ip_address,
            network_id=data.network_id,
        )
    except master_client.MasterUnavailableError:
        LOG.error(
            'Master unreachable during release of %s. '
            'Will be cleaned up during next sync.',
            data.ip_address,
        )

    _cache.remove(data.ip_address)
    return {'ip_address': data.ip_address, 'status': 'released'}


@app.get('/v1/health')
def health():
    master_ok = _master.ping()
    return {
        'status': 'healthy' if master_ok else 'degraded',
        'master_connected': master_ok,
        'last_sync': _cache.last_sync_time,
        'cache_size': _cache.size,
    }


def main():
    setup()
    uvicorn.run(
        app,
        host=_config.agent.bind_host,
        port=_config.agent.bind_port,
    )
```

### 5.5 Master Client (Agent -> Master)

```python
# eab_agent/master_client.py

import requests as http_requests
from oslo_log import log

LOG = log.getLogger(__name__)


class MasterUnavailableError(Exception):
    pass


class AddressConflictError(Exception):
    pass


class MasterClient:
    """Client for agent -> master communication."""

    def __init__(self, endpoint, auth_token, timeout, cloudpod_id,
                 max_retries=3, retry_interval=1):
        self._endpoint = endpoint
        self._timeout = timeout
        self._cloudpod_id = cloudpod_id
        self._max_retries = max_retries
        self._retry_interval = retry_interval
        self._session = http_requests.Session()
        self._session.headers.update({
            'Authorization': f'Bearer {auth_token}',
            'X-CloudPod-ID': cloudpod_id,
        })

    def _request(self, method, path, **kwargs):
        """Request with retry logic."""
        url = f'{self._endpoint}{path}'
        last_exc = None

        for attempt in range(self._max_retries):
            try:
                resp = self._session.request(
                    method, url, timeout=self._timeout, **kwargs,
                )
                return resp
            except (http_requests.ConnectionError,
                    http_requests.Timeout) as e:
                last_exc = e
                LOG.warning(
                    'Master request failed (attempt %d/%d): %s',
                    attempt + 1, self._max_retries, e,
                )
                if attempt < self._max_retries - 1:
                    import time
                    time.sleep(self._retry_interval)

        raise MasterUnavailableError(
            f'Master unreachable after {self._max_retries} attempts: '
            f'{last_exc}'
        )

    def reserve(self, network_id, subnet_id=None, project_id=None):
        payload = {
            'network_id': network_id,
            'cloudpod_id': self._cloudpod_id,
            'project_id': project_id,
        }
        if subnet_id:
            payload['subnet_id'] = subnet_id

        resp = self._request('POST', '/v1/addresses/reserve', json=payload)

        if resp.status_code == 200:
            return resp.json()
        elif resp.status_code == 409:
            raise AddressConflictError(resp.json().get('detail', ''))
        else:
            raise MasterUnavailableError(
                f'Unexpected response: {resp.status_code}'
            )

    def reserve_specific(self, network_id, ip_address, project_id=None):
        payload = {
            'network_id': network_id,
            'ip_address': ip_address,
            'cloudpod_id': self._cloudpod_id,
            'project_id': project_id,
        }
        resp = self._request('POST', '/v1/addresses/reserve', json=payload)

        if resp.status_code == 200:
            return resp.json()
        elif resp.status_code == 409:
            raise AddressConflictError(
                f'IP {ip_address} already allocated'
            )
        else:
            raise MasterUnavailableError(
                f'Unexpected response: {resp.status_code}'
            )

    def release(self, ip_address, network_id):
        payload = {
            'ip_address': ip_address,
            'network_id': network_id,
            'cloudpod_id': self._cloudpod_id,
        }
        resp = self._request('POST', '/v1/addresses/release', json=payload)

        if resp.status_code not in (200, 404):
            LOG.error('Error during release: %s %s',
                      resp.status_code, resp.text)

    def get_allocations(self):
        """Fetches all allocations for sync."""
        resp = self._request('GET', '/v1/addresses')
        if resp.status_code == 200:
            return resp.json().get('allocations', [])
        raise MasterUnavailableError(
            f'Sync failed: {resp.status_code}'
        )

    def heartbeat(self):
        resp = self._request('POST', '/v1/cloudpods/heartbeat', json={
            'cloudpod_id': self._cloudpod_id,
        })
        return resp.status_code == 200

    def ping(self):
        try:
            resp = self._request('GET', '/v1/health')
            return resp.status_code == 200
        except MasterUnavailableError:
            return False
```

### 5.6 Local Cache

```python
# eab_agent/cache.py

import datetime
import threading


class AllocationCache:
    """Thread-safe in-memory cache for IP allocations.

    The cache primarily serves as a quick overview and is updated
    during the full sync with the master.
    """

    def __init__(self):
        self._lock = threading.Lock()
        self._allocations = {}  # ip_address -> allocation_data
        self._last_sync = None

    def add(self, ip_address, data):
        with self._lock:
            self._allocations[ip_address] = data

    def remove(self, ip_address):
        with self._lock:
            self._allocations.pop(ip_address, None)

    def get(self, ip_address):
        with self._lock:
            return self._allocations.get(ip_address)

    def is_allocated(self, ip_address):
        with self._lock:
            return ip_address in self._allocations

    def full_sync(self, allocations):
        """Replaces the cache entirely with data from the master."""
        with self._lock:
            self._allocations = {
                a['ip_address']: a for a in allocations
            }
            self._last_sync = datetime.datetime.now(
                tz=datetime.timezone.utc,
            ).isoformat()

    @property
    def last_sync_time(self):
        return self._last_sync

    @property
    def size(self):
        with self._lock:
            return len(self._allocations)
```

## 6. Component: EAB Master (central)

The master is the single source of truth for all IP allocations across
all CloudPods.

### 6.1 Responsibilities

- Maintains the authoritative database of all allocations
- Accepts reserve/release requests from agents
- Ensures atomicity for concurrent reservations
- Manages registered CloudPods and their health status
- Provides sync endpoints
- Offers an admin API for manual interventions

### 6.2 Configuration (eab-master.conf)

```ini
[DEFAULT]
log_file = /var/log/eab/eab-master.log

[master]
# Bind address
bind_host = 0.0.0.0
bind_port = 9533

[database]
# SQLAlchemy connection string
connection = postgresql://eab:password@localhost:5432/eab_master

[auth]
# Comma-separated list of valid agent tokens
# In production: Keystone or another auth service
agent_tokens = token-cloudpod-a,token-cloudpod-b,token-cloudpod-c

[cleanup]
# Interval in seconds for checking orphaned allocations
orphan_check_interval = 300

# After how many seconds without heartbeat a CloudPod is considered offline
cloudpod_offline_timeout = 60
```

### 6.3 Data Model

```python
# eab_master/db/models.py

import datetime

from sqlalchemy import (
    Boolean, Column, DateTime, ForeignKey, Index, Integer,
    String, UniqueConstraint,
)
from sqlalchemy.orm import declarative_base, relationship

Base = declarative_base()


class CloudPod(Base):
    """A registered CloudPod."""
    __tablename__ = 'cloudpods'

    id = Column(String(64), primary_key=True)  # e.g. "cloudpod-a"
    name = Column(String(255), nullable=False)
    last_heartbeat = Column(DateTime(timezone=True), nullable=True)
    is_online = Column(Boolean, default=False, nullable=False)
    created_at = Column(
        DateTime(timezone=True),
        default=lambda: datetime.datetime.now(tz=datetime.timezone.utc),
    )

    allocations = relationship('Allocation', back_populates='cloudpod')


class ManagedSubnet(Base):
    """A subnet managed by the broker.

    References the subnet CIDRs configured on the external networks of the
    CloudPods. The neutron_network_id can differ per CloudPod, but the CIDR
    is identical.
    """
    __tablename__ = 'managed_subnets'

    id = Column(Integer, primary_key=True, autoincrement=True)
    cidr = Column(String(64), nullable=False, unique=True)
    name = Column(String(255), nullable=True)
    gateway_ip = Column(String(64), nullable=True)
    pool_start = Column(String(64), nullable=False)
    pool_end = Column(String(64), nullable=False)
    is_active = Column(Boolean, default=True, nullable=False)
    created_at = Column(
        DateTime(timezone=True),
        default=lambda: datetime.datetime.now(tz=datetime.timezone.utc),
    )

    allocations = relationship('Allocation', back_populates='subnet')


class NetworkMapping(Base):
    """Mapping from CloudPod-specific Neutron network/subnet IDs
    to managed subnets.

    Each CloudPod has its own UUIDs for its networks and subnets,
    but they reference the same physical network.
    """
    __tablename__ = 'network_mappings'

    id = Column(Integer, primary_key=True, autoincrement=True)
    cloudpod_id = Column(
        String(64), ForeignKey('cloudpods.id'), nullable=False,
    )
    neutron_network_id = Column(String(36), nullable=False)
    neutron_subnet_id = Column(String(36), nullable=True)
    managed_subnet_id = Column(
        Integer, ForeignKey('managed_subnets.id'), nullable=False,
    )

    __table_args__ = (
        UniqueConstraint(
            'cloudpod_id', 'neutron_network_id',
            name='uq_cloudpod_network',
        ),
    )

    cloudpod = relationship('CloudPod')
    managed_subnet = relationship('ManagedSubnet')


class Allocation(Base):
    """An IP address allocation."""
    __tablename__ = 'allocations'

    id = Column(Integer, primary_key=True, autoincrement=True)
    ip_address = Column(String(64), nullable=False)
    managed_subnet_id = Column(
        Integer, ForeignKey('managed_subnets.id'), nullable=False,
    )
    cloudpod_id = Column(
        String(64), ForeignKey('cloudpods.id'), nullable=False,
    )
    project_id = Column(String(36), nullable=True)
    reservation_id = Column(String(36), nullable=False, unique=True)
    allocated_at = Column(
        DateTime(timezone=True),
        default=lambda: datetime.datetime.now(tz=datetime.timezone.utc),
    )

    __table_args__ = (
        # An IP per subnet may only be allocated once
        UniqueConstraint(
            'ip_address', 'managed_subnet_id',
            name='uq_ip_per_subnet',
        ),
        Index('ix_alloc_cloudpod', 'cloudpod_id'),
        Index('ix_alloc_ip', 'ip_address'),
    )

    cloudpod = relationship('CloudPod', back_populates='allocations')
    subnet = relationship('ManagedSubnet', back_populates='allocations')
```

### 6.4 ER Diagram

```
+---------------+       +-------------------+       +----------------+
|  CloudPod     |       | NetworkMapping    |       | ManagedSubnet  |
+---------------+       +-------------------+       +----------------+
| id (PK)       |--+    | id (PK)           |   +-->| id (PK)        |
| name          |  +--->| cloudpod_id (FK)  |   |   | cidr           |
| last_heartbeat|  |    | neutron_network_id|   |   | pool_start     |
| is_online     |  |    | neutron_subnet_id |   |   | pool_end       |
+---------------+  |    | managed_subnet_id |---+   | is_active      |
                   |    +-------------------+       +--------+-------+
                   |                                         |
                   |    +--------------------+               |
                   |    | Allocation         |               |
                   |    +--------------------+               |
                   |    | id (PK)            |               |
                   +--->| cloudpod_id (FK)   |               |
                        | ip_address         |               |
                        | managed_subnet_id. |<--------------+
                        | project_id         |
                        | reservation_id     |
                        | allocated_at       |
                        |                    |
                        | UNIQUE(ip_address, |
                        |  managed_subnet_id)|
                        +--------------------+
```

### 6.5 Master API

```python
# eab_master/api/v1.py

import datetime
from collections.abc import Generator
from typing import Annotated

import netaddr
from fastapi import Depends, FastAPI, Header, HTTPException, Query
from oslo_config import cfg
from oslo_log import log
from oslo_utils import uuidutils
from pydantic import BaseModel
from sqlalchemy import select
from sqlalchemy.exc import IntegrityError
from sqlalchemy.orm import Session

from eab_master.db import models

LOG = log.getLogger(__name__)


# -----------------------------------------------------------------
# Pydantic models
# -----------------------------------------------------------------

class ReserveRequest(BaseModel):
    cloudpod_id: str
    network_id: str
    ip_address: str | None = None
    subnet_id: str | None = None
    project_id: str | None = None


class ReleaseRequest(BaseModel):
    ip_address: str
    network_id: str


class HeartbeatRequest(BaseModel):
    cloudpod_id: str


class CreateSubnetRequest(BaseModel):
    cidr: str
    name: str | None = None
    gateway_ip: str | None = None
    pool_start: str
    pool_end: str


class CreateCloudPodRequest(BaseModel):
    id: str
    name: str


class CreateMappingRequest(BaseModel):
    cloudpod_id: str
    neutron_network_id: str
    neutron_subnet_id: str | None = None
    managed_subnet_id: int


# -----------------------------------------------------------------
# App factory
# -----------------------------------------------------------------

def create_app(session_factory):
    app = FastAPI()

    def get_db() -> Generator[Session]:
        db = session_factory()
        try:
            yield db
        finally:
            db.close()

    def authenticate(
        authorization: Annotated[str, Header()] = '',
    ) -> str:
        """Validates the agent token."""
        token = authorization.replace('Bearer ', '')
        valid_tokens = cfg.CONF.auth.agent_tokens.split(',')
        if token not in valid_tokens:
            raise HTTPException(status_code=401, detail='Unauthorized')
        return token

    # -----------------------------------------------------------------
    # Address Management
    # -----------------------------------------------------------------

    @app.post('/v1/addresses/reserve')
    def reserve_address(
        data: ReserveRequest,
        db: Session = Depends(get_db),
        _token: str = Depends(authenticate),
    ):
        """Atomically reserves an IP address.

        Expected body:
            - network_id: Neutron network UUID of the requesting CloudPod
            - cloudpod_id: ID of the CloudPod
            - ip_address: (optional) specific IP
            - subnet_id: (optional) restriction to subnet
            - project_id: (optional) tenant/project
        """
        try:
            # Find mapping: which ManagedSubnet belongs to this
            # CloudPod + network combination?
            mapping = db.execute(
                select(models.NetworkMapping).where(
                    models.NetworkMapping.cloudpod_id == data.cloudpod_id,
                    models.NetworkMapping.neutron_network_id == data.network_id,
                )
            ).scalar_one_or_none()

            if not mapping:
                raise HTTPException(
                    status_code=404,
                    detail=f'No mapping found for CloudPod {data.cloudpod_id} '
                           f'and network {data.network_id}',
                )

            managed_subnet = db.get(
                models.ManagedSubnet, mapping.managed_subnet_id,
            )
            if not managed_subnet or not managed_subnet.is_active:
                raise HTTPException(
                    status_code=409, detail='Subnet is not active',
                )

            if data.ip_address:
                # Reserve specific IP
                alloc_ip = data.ip_address
            else:
                # Find next free IP
                alloc_ip = _find_free_ip(db, managed_subnet)
                if not alloc_ip:
                    raise HTTPException(
                        status_code=409,
                        detail='No free IP address available',
                    )

            reservation_id = uuidutils.generate_uuid()
            allocation = models.Allocation(
                ip_address=alloc_ip,
                managed_subnet_id=managed_subnet.id,
                cloudpod_id=data.cloudpod_id,
                project_id=data.project_id,
                reservation_id=reservation_id,
            )
            db.add(allocation)
            db.commit()

            return {
                'ip_address': alloc_ip,
                'subnet_id': mapping.neutron_subnet_id,
                'network_id': data.network_id,
                'reservation_id': reservation_id,
            }

        except IntegrityError:
            db.rollback()
            raise HTTPException(
                status_code=409,
                detail=f'IP {data.ip_address} is already allocated',
            )

        except HTTPException:
            raise

        except Exception:
            db.rollback()
            LOG.exception('Error during reservation')
            raise HTTPException(status_code=500, detail='Internal error')

    @app.post('/v1/addresses/release')
    def release_address(
        data: ReleaseRequest,
        x_cloudpod_id: Annotated[str, Header()],
        db: Session = Depends(get_db),
        _token: str = Depends(authenticate),
    ):
        """Releases an IP address."""
        try:
            allocation = db.execute(
                select(models.Allocation).where(
                    models.Allocation.ip_address == data.ip_address,
                    models.Allocation.cloudpod_id == x_cloudpod_id,
                )
            ).scalar_one_or_none()

            if not allocation:
                raise HTTPException(
                    status_code=404, detail='Allocation not found',
                )

            db.delete(allocation)
            db.commit()

            LOG.info(
                'IP %s released (CloudPod: %s)',
                data.ip_address, x_cloudpod_id,
            )
            return {
                'ip_address': data.ip_address,
                'status': 'released',
            }

        except HTTPException:
            raise

        except Exception:
            db.rollback()
            LOG.exception('Error during release')
            raise HTTPException(status_code=500, detail='Internal error')

    @app.get('/v1/addresses')
    def list_allocations(
        cloudpod_id: str | None = Query(default=None),
        db: Session = Depends(get_db),
        _token: str = Depends(authenticate),
    ):
        """Lists all allocations. Used by the agent for full sync.

        Optional query parameters:
            - cloudpod_id: Filter by CloudPod
        """
        query = select(models.Allocation)

        if cloudpod_id:
            query = query.where(
                models.Allocation.cloudpod_id == cloudpod_id,
            )

        allocations = db.execute(query).scalars().all()

        return {
            'allocations': [
                {
                    'ip_address': a.ip_address,
                    'cloudpod_id': a.cloudpod_id,
                    'managed_subnet_id': a.managed_subnet_id,
                    'project_id': a.project_id,
                    'reservation_id': a.reservation_id,
                    'allocated_at': a.allocated_at.isoformat(),
                }
                for a in allocations
            ],
        }

    # -----------------------------------------------------------------
    # CloudPod Management
    # -----------------------------------------------------------------

    @app.post('/v1/cloudpods/heartbeat')
    def cloudpod_heartbeat(
        data: HeartbeatRequest,
        db: Session = Depends(get_db),
        _token: str = Depends(authenticate),
    ):
        cloudpod = db.get(models.CloudPod, data.cloudpod_id)
        if not cloudpod:
            raise HTTPException(
                status_code=404, detail='CloudPod not registered',
            )

        cloudpod.last_heartbeat = datetime.datetime.now(
            tz=datetime.timezone.utc,
        )
        cloudpod.is_online = True
        db.commit()
        return {'status': 'ok'}

    @app.get('/v1/cloudpods')
    def list_cloudpods(
        db: Session = Depends(get_db),
        _token: str = Depends(authenticate),
    ):
        cloudpods = db.execute(select(models.CloudPod)).scalars().all()
        return {
            'cloudpods': [
                {
                    'id': c.id,
                    'name': c.name,
                    'is_online': c.is_online,
                    'last_heartbeat': (
                        c.last_heartbeat.isoformat()
                        if c.last_heartbeat else None
                    ),
                }
                for c in cloudpods
            ],
        }

    # -----------------------------------------------------------------
    # Admin: Subnet & Mapping Management
    # -----------------------------------------------------------------

    @app.get('/v1/admin/subnets')
    def list_managed_subnets(
        db: Session = Depends(get_db),
        _token: str = Depends(authenticate),
    ):
        subnets = db.execute(
            select(models.ManagedSubnet)
        ).scalars().all()
        return {
            'subnets': [
                {
                    'id': s.id,
                    'cidr': s.cidr,
                    'name': s.name,
                    'pool_start': s.pool_start,
                    'pool_end': s.pool_end,
                    'is_active': s.is_active,
                }
                for s in subnets
            ],
        }

    @app.post('/v1/admin/subnets', status_code=201)
    def create_managed_subnet(
        data: CreateSubnetRequest,
        db: Session = Depends(get_db),
        _token: str = Depends(authenticate),
    ):
        try:
            subnet = models.ManagedSubnet(
                cidr=data.cidr,
                name=data.name,
                gateway_ip=data.gateway_ip,
                pool_start=data.pool_start,
                pool_end=data.pool_end,
            )
            db.add(subnet)
            db.commit()
            return {'id': subnet.id, 'cidr': subnet.cidr}
        except IntegrityError:
            db.rollback()
            raise HTTPException(
                status_code=409,
                detail=f'Subnet {data.cidr} already exists',
            )

    @app.post('/v1/admin/cloudpods', status_code=201)
    def create_cloudpod(
        data: CreateCloudPodRequest,
        db: Session = Depends(get_db),
        _token: str = Depends(authenticate),
    ):
        try:
            cloudpod = models.CloudPod(
                id=data.id,
                name=data.name,
            )
            db.add(cloudpod)
            db.commit()
            return {'id': cloudpod.id, 'name': cloudpod.name}
        except IntegrityError:
            db.rollback()
            raise HTTPException(
                status_code=409,
                detail=f'CloudPod {data.id} already exists',
            )

    @app.post('/v1/admin/mappings', status_code=201)
    def create_network_mapping(
        data: CreateMappingRequest,
        db: Session = Depends(get_db),
        _token: str = Depends(authenticate),
    ):
        try:
            mapping = models.NetworkMapping(
                cloudpod_id=data.cloudpod_id,
                neutron_network_id=data.neutron_network_id,
                neutron_subnet_id=data.neutron_subnet_id,
                managed_subnet_id=data.managed_subnet_id,
            )
            db.add(mapping)
            db.commit()
            return {'id': mapping.id}
        except IntegrityError:
            db.rollback()
            raise HTTPException(
                status_code=409, detail='Mapping already exists',
            )

    @app.get('/v1/admin/mappings')
    def list_network_mappings(
        db: Session = Depends(get_db),
        _token: str = Depends(authenticate),
    ):
        mappings = db.execute(
            select(models.NetworkMapping)
        ).scalars().all()
        return {
            'mappings': [
                {
                    'id': m.id,
                    'cloudpod_id': m.cloudpod_id,
                    'neutron_network_id': m.neutron_network_id,
                    'neutron_subnet_id': m.neutron_subnet_id,
                    'managed_subnet_id': m.managed_subnet_id,
                }
                for m in mappings
            ],
        }

    # -----------------------------------------------------------------
    # Health
    # -----------------------------------------------------------------

    @app.get('/v1/health')
    def health():
        return {'status': 'healthy'}

    return app


def _find_free_ip(db, managed_subnet):
    """Finds the next free IP in the pool.

    Iterates over the IP range and checks against existing allocations.
    """
    allocated = {
        row.ip_address for row in db.execute(
            select(models.Allocation.ip_address).where(
                models.Allocation.managed_subnet_id == managed_subnet.id,
            )
        ).all()
    }

    pool = netaddr.IPRange(managed_subnet.pool_start, managed_subnet.pool_end)
    for ip in pool:
        ip_str = str(ip)
        if ip_str not in allocated:
            return ip_str

    return None
```

### 6.6 Master App Entry Point

```python
# eab_master/app.py

import uvicorn
from oslo_config import cfg
from oslo_log import log
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from eab_master.api import v1 as api_v1
from eab_master import config as master_config
from eab_master.db import models

LOG = log.getLogger(__name__)


def main():
    master_config.register_opts(cfg.CONF)
    cfg.CONF(project='eab-master')
    log.setup(cfg.CONF, 'eab-master')

    engine = create_engine(cfg.CONF.database.connection)
    models.Base.metadata.create_all(engine)
    session_factory = sessionmaker(bind=engine)

    app = api_v1.create_app(session_factory)

    LOG.info(
        'EAB Master starting on %s:%s',
        cfg.CONF.master.bind_host,
        cfg.CONF.master.bind_port,
    )

    uvicorn.run(
        app,
        host=cfg.CONF.master.bind_host,
        port=cfg.CONF.master.bind_port,
    )
```

## 7. Sequence Diagrams

### 7.1 Create Floating IP (automatic IP)

```
User                Neutron              IPAM Driver          EAB Agent         EAB Master
 |                    |                     |                    |                  |
 | create_floatingip  |                     |                    |                  |
 |------------------->|                     |                    |                  |
 |                    | create_port         |                    |                  |
 |                    | (FLOATINGIP owner)  |                    |                  |
 |                    |-------------------->|                    |                  |
 |                    |                     |                    |                  |
 |                    |          get_request(port, ip_dict)      |                  |
 |                    |                     |                    |                  |
 |                    |                     | device_owner ==    |                  |
 |                    |                     | FLOATINGIP?        |                  |
 |                    |                     | YES                |                  |
 |                    |                     |                    |                  |
 |                    |                     | POST /v1/addresses/|reserve           |
 |                    |                     |------------------->|                  |
 |                    |                     |                    | POST /v1/        |
 |                    |                     |                    | addresses/reserve|
 |                    |                     |                    |----------------->|
 |                    |                     |                    |                  |
 |                    |                     |                    |    Atomic:       |
 |                    |                     |                    |    find free IP, |
 |                    |                     |                    |    INSERT        |
 |                    |                     |                    |    allocation    |
 |                    |                     |                    |                  |
 |                    |                     |                    | 200 {ip, subnet} |
 |                    |                     |                    |<-----------------|
 |                    |                     | 200 {ip, subnet}   |                  |
 |                    |                     |<-------------------|                  |
 |                    |                     |                    |                  |
 |                    |  SpecificAddressRequest(ip)              |                  |
 |                    |<--------------------|                    |                  |
 |                    |                     |                    |                  |
 |                    | allocate(request)   |                    |                  |
 |                    | (NeutronDB local)   |                    |                  |
 |                    |                     |                    |                  |
 | 201 floatingip     |                     |                    |                  |
 |<-------------------|                     |                    |                  |
```

### 7.2 Delete Floating IP

```
User                Neutron              Callback             EAB Agent         EAB Master
 |                    |                     |                    |                  |
 | delete_floatingip  |                     |                    |                  |
 |------------------->|                     |                    |                  |
 |                    | delete FIP + port   |                    |                  |
 |                    | (local in DB)       |                    |                  |
 |                    |                     |                    |                  |
 |                    | AFTER_DELETE event  |                    |                  |
 |                    |-------------------->|                    |                  |
 |                    |                     | POST /v1/addresses/|release           |
 |                    |                     |------------------->|                  |
 |                    |                     |                    | POST /v1/        |
 |                    |                     |                    | addresses/release|
 |                    |                     |                    |----------------->|
 |                    |                     |                    |                  |
 |                    |                     |                    |  DELETE alloc    |
 |                    |                     |                    |                  |
 |                    |                     |                    | 200              |
 |                    |                     |                    |<-----------------|
 |                    |                     | 200                |                  |
 |                    |                     |<-------------------|                  |
 | 204                |                     |                    |                  |
 |<-------------------|                     |                    |                  |
```

### 7.3 Periodic Sync (Agent -> Master)

```
EAB Agent                                EAB Master
 |                                          |
 |  GET /v1/addresses?cloudpod_id=a         |
 |----------------------------------------->|
 |                                          |
 |  200 {allocations: [...]}                |
 |<-----------------------------------------|
 |                                          |
 |  Update cache                            |
 |  (full_sync)                             |
 |                                          |
 |  Compare: local Neutron FIPs vs          |
 |  master allocations                      |
 |                                          |
 |  If discrepancy:                         |
 |  - IP in Neutron but not in master       |
 |    -> log warning (manual intervention)  |
 |  - IP in master but not in Neutron       |
 |    -> send release to master             |
 |                                          |
```

## 8. Error Scenarios and Handling

### 8.1 Master Unreachable During Reserve

```
Behavior depends on configuration:

fallback_to_local = false (default, recommended):
    -> AgentUnavailableError
    -> Neutron returns 503 to the user
    -> No duplicate allocation possible
    -> User must retry later

fallback_to_local = true (emergency only):
    -> IPAM driver falls back to AnyAddressRequest
    -> Local allocation without master check
    -> WARNING: Duplicate allocations possible!
    -> Discrepancy detected during next sync
```

### 8.2 Agent Unreachable

```
IPAM driver cannot reach the local agent:
    -> Same behavior as "master unreachable"
    -> The agent runs locally, so this should almost never happen
    -> systemd/supervisor ensures it is running
```

### 8.3 Race Condition: Two CloudPods Reserve Simultaneously

```
CloudPod A                   EAB Master                    CloudPod B
    |                            |                              |
    | reserve(auto)              |                reserve(auto) |
    |--------------------------->|<-----------------------------|
    |                            |                              |
    |                      +-----+------+                       |
    |                      | Serialized |                       |
    |                      | via DB     |                       |
    |                      | UNIQUE     |                       |
    |                      | constraint |                       |
    |                      +-----+------+                       |
    |                            |                              |
    | 200 {ip: .42}              |              200 {ip: .43}   |
    |<---------------------------|----------------------------->|

Atomicity is guaranteed by the UNIQUE constraint
(ip_address, managed_subnet_id) in the master DB.
For automatic allocation: _find_free_ip() + INSERT within a
single DB transaction. On IntegrityError: retry with next free IP.
```

### 8.4 Orphaned Allocations

```
Scenario: Neutron deletes a FIP, but the AFTER_DELETE callback
          cannot reach the agent/master.

Solution: Periodic sync in the agent:
    1. Agent fetches all its own allocations from master
    2. Agent checks against local Neutron DB (FIPs)
    3. Allocations that exist in master but not in Neutron
       -> send release to master

Additionally: Master-side orphan check:
    - Configurable via cleanup.orphan_check_interval
    - Master can ask agents about the status of an allocation
```

### 8.5 CloudPod Goes Offline

```
Scenario: A CloudPod fails completely.

Behavior:
    - Heartbeat stops
    - Master marks CloudPod as offline after cloudpod_offline_timeout
    - Allocations of the CloudPod remain in place (IPs stay reserved)
    - IPs are NOT automatically released
      (the CloudPod might still be announcing BGP routes)
    - Manual release via admin API once it is confirmed
      that the CloudPod is no longer using the IPs
```

## 9. Setting Up a New Environment

Step-by-step guide for adding a new CloudPod.

### 9.1 Prerequisites

- EAB master is running and reachable
- New CloudPod has an external network with the same CIDRs

### 9.2 On the Master

```bash
# 1. Register CloudPod
curl -X POST https://eab-master:9533/v1/admin/cloudpods \
    -H "Authorization: Bearer $ADMIN_TOKEN" \
    -d '{
        "id": "cloudpod-c",
        "name": "CloudPod Berlin 1"
    }'

# 2. Create managed subnet (if not already present)
curl -X POST https://eab-master:9533/v1/admin/subnets \
    -H "Authorization: Bearer $ADMIN_TOKEN" \
    -d '{
        "cidr": "198.51.100.0/24",
        "name": "Public IPv4",
        "gateway_ip": "198.51.100.1",
        "pool_start": "198.51.100.10",
        "pool_end": "198.51.100.254"
    }'

# 3. Create network mapping
#    (Map Neutron UUIDs of the new CloudPod to the ManagedSubnet)
curl -X POST https://eab-master:9533/v1/admin/mappings \
    -H "Authorization: Bearer $ADMIN_TOKEN" \
    -d '{
        "cloudpod_id": "cloudpod-c",
        "neutron_network_id": "uuid-of-external-net-in-cloudpod-c",
        "neutron_subnet_id": "uuid-of-subnet-in-cloudpod-c",
        "managed_subnet_id": 1
    }'
```

### 9.3 On the New CloudPod

```bash
# 1. Install plugin
pip install networking-external-address-broker

# 2. Configure neutron.conf
cat >> /etc/neutron/neutron.conf <<EOF

[DEFAULT]
ipam_driver = external_address_broker

[external_address_broker]
agent_endpoint = http://localhost:9532
agent_timeout = 5
fallback_to_local = false
EOF

# 3. Configure EAB agent
cat > /etc/eab/eab-agent.conf <<EOF
[agent]
cloudpod_id = cloudpod-c
cloudpod_name = CloudPod Berlin 1
bind_host = 127.0.0.1
bind_port = 9532

[master]
endpoint = https://eab-master.example.com:9533
auth_token = token-cloudpod-c
timeout = 10

[sync]
full_sync_interval = 60
heartbeat_interval = 10
EOF

# 4. Start agent
systemctl enable --now eab-agent

# 5. Restart Neutron server
systemctl restart neutron-server
```

## 10. Monitoring and Operations

### 10.1 Key Metrics

| Metric | Source | Alert when |
|--------|--------|------------|
| Master reachable | Agent /v1/health | master_connected == false |
| Agent reachable | Neutron logs | AgentUnavailableError |
| Sync age | Agent /v1/health | last_sync > 5 minutes |
| Free IPs | Master DB | < 10% of pool free |
| CloudPod offline | Master DB | is_online == false |
| Allocation discrepancy | Agent sync log | Warning in log |

### 10.2 Log Messages

```
# Normal operation
EAB: IP 198.51.100.42 reserved for network <uuid>
EAB: Releasing IP 198.51.100.42 on network <uuid>
Full sync completed, 42 entries

# Warnings
EAB agent unreachable, falling back to local allocation
Master unreachable during release of 198.51.100.42
Heartbeat to master failed

# Errors
EAB agent unreachable: http://localhost:9532
Address must be released manually
```

### 10.3 Admin Commands

```bash
# List all allocations
curl https://eab-master:9533/v1/addresses \
    -H "Authorization: Bearer $TOKEN"

# Allocations for a specific CloudPod
curl "https://eab-master:9533/v1/addresses?cloudpod_id=cloudpod-a" \
    -H "Authorization: Bearer $TOKEN"

# All CloudPods and their status
curl https://eab-master:9533/v1/cloudpods \
    -H "Authorization: Bearer $TOKEN"

# Managed subnets
curl https://eab-master:9533/v1/admin/subnets \
    -H "Authorization: Bearer $TOKEN"

# Network mappings
curl https://eab-master:9533/v1/admin/mappings \
    -H "Authorization: Bearer $TOKEN"

# Manual IP release (e.g. after CloudPod failure)
curl -X POST https://eab-master:9533/v1/addresses/release \
    -H "Authorization: Bearer $TOKEN" \
    -H "X-CloudPod-ID: cloudpod-a" \
    -d '{"ip_address": "198.51.100.42", "network_id": "n/a"}'
```

## 11. Security

### 11.1 Communication

- Agent <-> Master: TLS (HTTPS) with token authentication
- Neutron <-> Agent: Local (localhost), no TLS needed
- Master API: Only reachable by agents and admins (firewall/network)

### 11.2 Authentication

- Initial implementation: Shared secrets (bearer token per CloudPod)
- Extension: Keystone service tokens or mTLS between agent and master

### 11.3 Authorization

- Agents can only reserve/release addresses for their own CloudPod
  (enforced by X-CloudPod-ID header + token validation)
- Admin endpoints (/v1/admin/*) require a separate admin token

## 12. Open Items and Extensions

### 12.1 Phase 1 (MVP)

- [x] IPAM driver with AddressRequestFactory
- [x] Agent with reserve/release API
- [x] Master with atomic reservation
- [x] Periodic sync
- [x] Network mappings

### 12.2 Phase 2

- [ ] High availability: Master behind load balancer with
      PostgreSQL replication
- [ ] Rate limiting on the master
- [ ] Audit log for all allocations/releases
- [ ] CLI tool for admin operations (eab-manage)
- [ ] Prometheus metrics endpoint on agent and master

### 12.3 Phase 3

- [ ] Keystone integration for authentication
- [ ] Automatic subnet discovery (agent detects external networks
      and registers mappings automatically at the master)
- [ ] Web dashboard for allocation overview
- [ ] Webhook notifications on allocation changes
      (e.g. for BGP controller integration)
- [ ] IPv6 support
- [ ] Quota management per CloudPod or project
