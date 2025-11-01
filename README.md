# ChirpStack for OptixEdge

Deploy [ChirpStack](https://www.chirpstack.io) LoRaWAN Network Server (v4) on Rockwell Automation OptixEdge devices using Portainer.
 
## Quick Start

1. In Portainer, navigate to **Stacks** → **Add stack**
2. Name your stack (e.g., `chirpstack`)
3. Choose one of:
   - **Repository**: `https://github.com/asqi-carter/chirpstack-docker-Optix-Edge`
   - **Web editor**: Paste the contents of `docker-compose.yml`
4. Add optional env vars seen in .env.example
5. Click **Deploy the stack**
6. Wait for `config-init` to complete (shows "Exited" status)
7. chirpstack-service will have some mqtt log errors on first deployment. stop and start the Portainer stack after first deployment to resolve
8. Access ChirpStack at `http://<optixedge-ip>:8080`

**Default credentials:**
- Username: `admin`
- Password: `admin`

## How This Works

This setup is optimized for OptixEdge devices, which restrict direct host filesystem access through Portainer.

Traditional ChirpStack Docker deployments mount configuration files directly from the host. Since OptixEdge doesn't allow this, we use a different approach:

**The `config-init` service:**
1. Runs once during deployment
2. Clones the official ChirpStack Docker repository
3. Copies all configuration files to Docker volumes
4. Auto-generates your API secret
5. Exits (showing "Exited" status is normal)

This gives you:
- All upstream ChirpStack configurations automatically
- No manual file management
- Works within OptixEdge/Portainer constraints
- Stays synchronized with official releases

Configuration files persist in Docker volumes and are reused on subsequent starts.

## Configuration (Optional)

All env vars are optional.

To customize, add environment variables in Portainer when deploying the stack:

| Variable | Default | Description |
|----------|---------|-------------|
| `CHIRPSTACK_API_SECRET` | auto-generated | Override the auto-generated API secret |
| `CHIRPSTACK_REGION` | `eu868` | LoRaWAN region for gateway bridge |
| `CHIRPSTACK_NETWORK_ID` | `000000` | LoRaWAN Network ID (3 bytes hex) |
| `CHIRPSTACK_HTTP_PORT` | `8080` | Web UI port |
| `GATEWAY_UDP_PORT` | `1700` | Semtech UDP gateway port |
| `GATEWAY_BASICSTATION_PORT` | `3001` | BasicStation gateway port |

### Supported Regions

| Region | Description |
|--------|-------------|
| `eu868` | Europe 863-870 MHz |
| `us915_0` - `us915_7` | United States (sub-bands) |
| `au915_0` - `au915_7` | Australia (sub-bands) |
| `as923`, `as923_2`, `as923_3`, `as923_4` | Asia Pacific |
| `cn470_0` - `cn470_11` | China (sub-bands) |
| `in865` | India |
| `kr920` | South Korea |
| `ru864` | Russia |
| `cn779` | China 779 MHz |
| `eu433` | Europe 433 MHz |
| `ism2400` | ISM 2.4 GHz |

## Connect a Gateway

### Semtech UDP Packet Forwarder

Configure your gateway with:
- **Server address**: `<optixedge-ip>`
- **Server port**: `1700`

Example gateway configuration:
```json
{
  "gateway_conf": {
    "server_address": "192.168.1.100",
    "serv_port_up": 1700,
    "serv_port_down": 1700
  }
}
```

### ChirpStack MQTT Forwarder

If your gateway runs ChirpStack Gateway Bridge or MQTT Forwarder:
```toml
[integration.mqtt]
server="tcp://<optixedge-ip>:1883"
```

Ensure the `topic_prefix` matches your `CHIRPSTACK_REGION` setting.

## Importing device repository

Import pre-configured device profiles from The Things Network's lorawan-devices repository:

1. In Portainer, go to **Containers**
2. Click `chirpstack-server`
3. Click **Console** → select `/bin/sh` as command and `root` as user
4. Click **Connect**
5. Run:
```bash
apk add --no-cache git && \
git clone https://github.com/brocaar/lorawan-devices /tmp/lorawan-devices && \
chirpstack -c /etc/chirpstack import-legacy-lorawan-devices-repository -d /tmp/lorawan-devices
```
6. Wait 5-10 minutes for import to complete
7. Device profile templates will appear in the ChirpStack web UI

This will clone the lorawan-devices repository and execute the import command of ChirpStack.

Note: an older snapshot of the lorawan-devices repository is cloned as the latest revision no longer contains a LICENSE file.

## Troubleshooting

### Stack Won't Start

Check the `config-init` container logs for errors. Ensure:
- OptixEdge has internet access to clone the git repository
- All previous stack containers are stopped

### MQTT Connection Errors

If you see MQTT connection errors in ChirpStack logs after first deployment:

1. In Portainer, go to **Stacks**
2. Click your stack name
3. Click **Stop** → wait for all containers to stop
4. Click **Start**

This timing issue typically resolves after one restart.

### Reset to Defaults

To completely reset and redeploy:

1. In Portainer, delete the stack
2. Go to **Volumes** → delete all volumes starting with your stack name
3. Redeploy the stack

Warning: This deletes all data including devices, gateways, and your API secret.

## Architecture

This deployment includes:

| Service | Description | Port |
|---------|-------------|------|
| chirpstack-postgres | PostgreSQL 14 database | Internal |
| chirpstack-redis | Redis 7 cache | Internal |
| chirpstack-mosquitto | MQTT broker | 1883 |
| chirpstack-server | ChirpStack network server | 8080 |
| chirpstack-gateway-bridge | Semtech UDP gateway bridge | 1700/udp |
| chirpstack-gateway-bridge-bs | BasicStation gateway bridge | 3001 |
| chirpstack-rest-api | REST API interface | 8090 |
| config-init | One-time initialization | - |

The `config-init` service clones configuration files from the official ChirpStack repository on first deployment and exits (normal behavior).

## Links

- [ChirpStack Documentation](https://www.chirpstack.io/)
- [ChirpStack Community Forum](https://forum.chirpstack.io/)
- [Original ChirpStack Docker Repository](https://github.com/chirpstack/chirpstack-docker)
- [OptixEdge Documentation](https://www.rockwellautomation.com/en-us/products/software/factorytalk/optixedge.html)

## License

ChirpStack is licensed under the MIT License. See the [ChirpStack LICENSE](https://github.com/chirpstack/chirpstack/blob/master/LICENSE) for details.
