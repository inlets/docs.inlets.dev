# High throughput tunnels

Use this pattern when a customer has several long-lived TCP services and you
want a single inlets uplink client process to carry them.

The same approach also works well for RDP, VNC, SSH, Postgres, Redis, and other
long-lived TCP connections. We will use RTSP for this tutorial.

With [demux](/uplink/troubleshooting/#demux-mode) enabled, inlets-pro opens a
separate websocket for each incoming TCP connection.

This tutorial runs three [MediaMTX](https://github.com/bluenviron/mediamtx)
instances on a private host and exposes their
[RTSP](https://en.wikipedia.org/wiki/Real-Time_Streaming_Protocol) streams
through one Uplink tunnel.

```text
+--------------------------------------+    +--------------------------------------+
| Private host                         |    | Uplink cluster                       |
|                                      |    |                                      |
| MediaMTX #1  rtsp://127.0.0.1:8551   |    | Service: mediamtx.streams.svc        |
| MediaMTX #2  rtsp://127.0.0.1:8552   |    | Ports:   8551, 8552, 8553            |
| MediaMTX #3  rtsp://127.0.0.1:8553   |    |                                      |
|                                      |    | Cluster clients or port-forward      |
|          \      |      /             |    |                                      |
|           \     |     /              |    | rtsp://mediamtx.streams:8551/stream  |
|       inlets-pro client --demux      |===>| rtsp://mediamtx.streams:8552/stream  |
|                                      |    | rtsp://mediamtx.streams:8553/stream  |
+--------------------------------------+    +--------------------------------------+
```

For testing, we will use a sample `demo.mp4` video file instead of a real
camera feed.

## Prerequisites

- Inlets Uplink installed in your cluster
- `inlets-pro` 0.11.11 or newer on the private host
- `mediamtx` and `ffmpeg` on the private host
- The tunnel plugin for the `inlets-pro` CLI

Install the tunnel plugin:

```bash
inlets-pro plugin get tunnel
```

You can install the host tools with
[arkade](https://github.com/alexellis/arkade), or via your package manager:

```bash
arkade get inlets-pro
arkade get mediamtx
```

Set the values used below:

```bash
export NS="streams"
export TUNNEL="mediamtx"
export DOMAIN="uplink.example.com"
```

## Create the tunnel

Create one Tunnel resource with three TCP ports.

Demux is enabled on the tunnel server with `uplink_demux=1`.

```yaml
apiVersion: uplink.inlets.dev/v1alpha1
kind: Tunnel
metadata:
  name: mediamtx
  namespace: streams
spec:
  licenseRef:
    name: inlets-uplink-license
    namespace: streams
  env:
    uplink_demux: "1"
  tcpPorts:
  - 8551
  - 8552
  - 8553
```

Apply it:

```bash
kubectl apply -f mediamtx-tunnel.yaml
kubectl get -n $NS tunnel/$TUNNEL
```

Wait for the tunnel server to start:

```bash
kubectl rollout status -n $NS deploy/$TUNNEL
```

## Run MediaMTX

On the private host, create a config directory:

```bash
mkdir -p ~/mediamtx-rtsp
```

Create three MediaMTX config files. Each instance listens on a different RTSP
port and has every other protocol disabled.

```bash
for i in 1 2 3; do
  port=$((8550+i))

  cat > ~/mediamtx-rtsp/rtsp-$i.yml <<EOF
logLevel: info
rtsp: yes
rtspTransports: [tcp]
rtspAddress: :$port
rtmp: no
hls: no
webrtc: no
srt: no
paths:
  stream:
    runOnInit: ffmpeg -hide_banner -loglevel warning -re -stream_loop -1 -i ./demo.mp4 -c:v libx264 -preset ultrafast -tune zerolatency -g 30 -keyint_min 30 -c:a aac -rtsp_transport tcp -f rtsp rtsp://127.0.0.1:$port/stream
    runOnInitRestart: yes
EOF
done
```

Replace `./demo.mp4` with your own video file.

Start the three servers:

```bash
cd ~/mediamtx-rtsp

mediamtx rtsp-1.yml &
mediamtx rtsp-2.yml &
mediamtx rtsp-3.yml &
```

Check that each stream is available locally:

```bash
for port in 8551 8552 8553; do
  ffprobe -v error \
    -rtsp_transport tcp \
    -select_streams v:0 \
    -show_entries stream=codec_name \
    -of default=nw=1:nk=1 \
    rtsp://127.0.0.1:$port/stream
done
```

You should see `h264` three times.

## Connect the tunnel client

Get the tunnel token:

```bash
inlets-pro tunnel token $TUNNEL \
  --namespace $NS > token.txt
```

You can also use the tunnel plugin to print the client command:

```bash
inlets-pro tunnel connect $TUNNEL \
  --namespace $NS \
  --domain $DOMAIN \
  --upstream 8551=127.0.0.1:8551 \
  --upstream 8552=127.0.0.1:8552 \
  --upstream 8553=127.0.0.1:8553 \
  --quiet
```

It prints the `inlets-pro uplink client` command for the tunnel:

```bash
inlets-pro uplink client \
  --url=wss://uplink.example.com/streams/mediamtx \
  --token=<redacted> \
  --upstream=8551=127.0.0.1:8551 \
  --upstream=8552=127.0.0.1:8552 \
  --upstream=8553=127.0.0.1:8553
```

Run one inlets-pro client with three TCP upstreams. Add `--demux` for this
RTSP use-case. It is optional, but highly recommended for multiple viewers or
long-lived TCP connections over the same tunnel.

```bash
inlets-pro uplink client \
  --url wss://$DOMAIN/$NS/$TUNNEL \
  --token-file ./token.txt \
  --upstream 8551=127.0.0.1:8551 \
  --upstream 8552=127.0.0.1:8552 \
  --upstream 8553=127.0.0.1:8553 \
  --demux
```

The left side of each `--upstream` is the tunnel port in Kubernetes. The right
side is the private host address.

For a host service, the same plugin can generate a systemd unit:

```bash
inlets-pro tunnel connect $TUNNEL \
  --namespace $NS \
  --domain $DOMAIN \
  --upstream 8551=127.0.0.1:8551 \
  --upstream 8552=127.0.0.1:8552 \
  --upstream 8553=127.0.0.1:8553 \
  --format systemd > $TUNNEL.service
```

Add `--demux` to the generated `ExecStart` line before installing the unit.

If the private MediaMTX service is running inside Kubernetes or K3s, generate a
client Deployment instead:

```bash
inlets-pro tunnel connect $TUNNEL \
  --namespace $NS \
  --domain $DOMAIN \
  --upstream 8551=mediamtx.default.svc.cluster.local:8551 \
  --upstream 8552=mediamtx.default.svc.cluster.local:8552 \
  --upstream 8553=mediamtx.default.svc.cluster.local:8553 \
  --format k8s_yaml \
  --inlets-version 0.11.11 > $TUNNEL-client.yaml
```

Add `--demux` to the generated client args, then apply the file.

## Test from the cluster

Create a temporary probe pod:

```bash
kubectl run -n $NS rtsp-probe \
  --image=alpine:latest \
  --restart=Never \
  --command -- sleep 3600

kubectl exec -n $NS rtsp-probe -- \
  apk add --no-cache ffmpeg
```

Probe all three streams through the same tunnel:

```bash
for port in 8551 8552 8553; do
  kubectl exec -n $NS rtsp-probe -- \
    ffprobe -v error \
      -rtsp_transport tcp \
      -select_streams v:0 \
      -show_entries stream=codec_name \
      -of default=nw=1:nk=1 \
      rtsp://$TUNNEL.$NS.svc.cluster.local:$port/stream
done
```

You should see `h264` for each port.

## View with port-forward

The Service is cluster-local. To view the streams from your laptop, forward the
three service ports:

```bash
kubectl port-forward -n $NS svc/$TUNNEL \
  8551:8551 \
  8552:8552 \
  8553:8553
```

Then open any of the streams with VLC, ffplay, or ffprobe:

```bash
ffplay -rtsp_transport tcp rtsp://127.0.0.1:8551/stream
ffplay -rtsp_transport tcp rtsp://127.0.0.1:8552/stream
ffplay -rtsp_transport tcp rtsp://127.0.0.1:8553/stream
```

If a local port is already in use, change only the left-hand side:

```bash
kubectl port-forward -n $NS svc/$TUNNEL 18551:8551
ffplay -rtsp_transport tcp rtsp://127.0.0.1:18551/stream
```

## Notes

- Use `rtspTransports: [tcp]` when running several MediaMTX instances on one
  host. Otherwise each instance also tries to bind the default RTP and RTCP UDP
  ports.
- Demux is useful for RTSP because every viewer gets its own websocket through
  the tunnel.
- The tunnel remains cluster-local unless you expose it with a Service,
  Ingress, or LoadBalancer.
