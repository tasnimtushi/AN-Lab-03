# Lab 3 : Secure Voice Model over IPSec Tunnel

## Introduction

Voice data can contain sensitive and private information. Sending audio directly over a public or untrusted network can expose the data to unauthorized access or interception. Similarly, an ML model serving endpoint does not always need to be publicly accessible. In a secure ML deployment, it is often better to keep the model server inside a private network and allow only authorized clients to communicate with it.

In this lab, you will build a simple **secure voice inference system** using an **IPSec tunnel**, **FastAPI**, and the **Whisper speech-to-text model**.

The client VM will generate and send an audio file to a private server. The communication between the client and server will pass through an encrypted IPSec tunnel. On the server side, FastAPI will receive the audio file and send it to the Whisper model for transcription. The transcription result will then be returned to the client through the same secure connection.

The overall architecture is:
![alt text](<lab-03 diagram-1.png>)


Throughout the lab, you will configure the IPSec tunnel using **strongSwan**, deploy the Whisper model as a FastAPI service, send audio through the private network, and inspect the network traffic using `tcpdump`.

This lab will help you connect three important areas of an ML deployment:

```text
Networking
    +
Security
    +
Machine Learning Model Serving
```

---

## Learning Objectives

By the end of this lab, you will be able to:

- Configure an IPSec tunnel using strongSwan.
- Understand the basic role of IKE and ESP in IPSec.
- Configure private IP communication between two VMs.
- Deploy a Whisper speech-to-text model using FastAPI.
- Send an audio file to a private ML inference endpoint.
- Test the FastAPI service from a client VM.
- Inspect IPSec traffic using `tcpdump`.
- Understand how encryption protects ML inference traffic.
- Keep an ML model serving endpoint inside a private network.



# Step 1 — Prepare the VMs

Use two Ubuntu VMs.

| VM | Transport IP | Private IP |
|---|---|---|
| Client | `192.168.56.101` | `10.10.1.10` |
| Server | `192.168.56.102` | `10.10.2.10` |

Check the IP addresses on both VMs:

```bash
ip addr
```

Test basic connectivity from the **Client VM**:

```bash
ping -c 3 192.168.56.102
```

Expected result:

```text
64 bytes from 192.168.56.102 ...
```

If the ping succeeds, the two VMs can communicate over their transport network.

---

# Step 2 — Create Private IPs

The private IPs will be used as the protected endpoints for the IPSec tunnel.

## Client VM

Run:

```bash
sudo ip addr add 10.10.1.10/32 dev lo
```

## Server VM

Run:

```bash
sudo ip addr add 10.10.2.10/32 dev lo
```

Verify the private IPs:

```bash
ip addr show lo
```

You should see the configured private IP address.

---

# Step 3 — Install strongSwan

`strongSwan` is an open-source implementation of IPSec. It will be used to establish the encrypted tunnel between the two VMs.

Run the following command on **both VMs**:

```bash
sudo apt update
sudo apt install -y strongswan
```

Check the installation:

```bash
ipsec --version
```

You should see the installed strongSwan version.

---

# Step 4 — Configure IPSec

In this step, you will configure an IKEv2-based IPSec tunnel using a pre-shared key (PSK).

## Client VM

Open the IPSec configuration file:

```bash
sudo nano /etc/ipsec.conf
```

Add the following configuration:

```conf
config setup

conn secure-voice
    type=tunnel
    keyexchange=ikev2
    authby=psk

    left=192.168.56.101
    leftsubnet=10.10.1.0/24

    right=192.168.56.102
    rightsubnet=10.10.2.0/24

    ike=aes256-sha256-modp2048
    esp=aes256-sha256

    auto=start
```

Here:

- `keyexchange=ikev2` uses IKEv2 for key exchange.
- `authby=psk` uses a pre-shared key for authentication.
- `left` represents the local transport IP.
- `right` represents the remote transport IP.
- `leftsubnet` defines the local private network.
- `rightsubnet` defines the remote private network.
- `ike` defines the IKE encryption and authentication algorithms.
- `esp` defines the ESP encryption and authentication algorithms.
- `auto=start` tells strongSwan to start the connection automatically.

## Server VM

Open the configuration file:

```bash
sudo nano /etc/ipsec.conf
```

Use the same configuration, but swap the `left` and `right` values:

```conf
config setup

conn secure-voice
    type=tunnel
    keyexchange=ikev2
    authby=psk

    left=192.168.56.102
    leftsubnet=10.10.2.0/24

    right=192.168.56.101
    rightsubnet=10.10.1.0/24

    ike=aes256-sha256-modp2048
    esp=aes256-sha256

    auto=start
```

---

# Step 5 — Configure the PSK

Both IPSec endpoints need the same pre-shared key.

On **both VMs**, open:

```bash
sudo nano /etc/ipsec.secrets
```

Add:

```conf
: PSK "PoridhiSecureVoice2026!"
```

Protect the secrets file:

```bash
sudo chmod 600 /etc/ipsec.secrets
```

Restart the IPSec service:

```bash
sudo systemctl restart strongswan-starter
```

Check the tunnel status:

```bash
sudo ipsec statusall
```

Look for an established:

- **IKE SA**
- **CHILD SA**

If they are established, the IPSec tunnel has been successfully negotiated.

---

# Step 6 — Check IPSec Policies

Linux uses **XFRM** to implement IPSec policies and states.

Check the IPSec state:

```bash
sudo ip xfrm state
```

Then check the policies:

```bash
sudo ip xfrm policy
```

The output should contain policies related to the private networks:

```text
10.10.1.0/24
10.10.2.0/24
```

These policies determine which traffic should be protected by IPSec.

---

# Step 7 — Install Whisper

Now you will deploy the speech-to-text model on the **Server VM**.

Install the required system packages:

```bash
sudo apt install -y python3-venv ffmpeg
```

Create a project directory:

```bash
mkdir ~/whisper-api
cd ~/whisper-api
```

Create a Python virtual environment:

```bash
python3 -m venv venv
```

Activate it:

```bash
source venv/bin/activate
```

Install the required Python packages:

```bash
pip install fastapi uvicorn python-multipart faster-whisper
```

---

# Step 8 — Create the FastAPI Server

Create the application file:

```bash
nano app.py
```

Add the following code:

```python
from fastapi import FastAPI, UploadFile, File
from faster_whisper import WhisperModel
import tempfile

app = FastAPI()

model = WhisperModel(
    "tiny",
    device="cpu",
    compute_type="int8"
)

@app.get("/health")
def health():
    return {"status": "healthy"}

@app.post("/transcribe")
async def transcribe(file: UploadFile = File(...)):

    with tempfile.NamedTemporaryFile(
        suffix=".wav"
    ) as temp:

        temp.write(await file.read())
        temp.flush()

        segments, info = model.transcribe(temp.name)

        text = " ".join(
            segment.text.strip()
            for segment in segments
        )

    return {
        "language": info.language,
        "text": text
    }
```

### What does this application do?

The FastAPI application provides two endpoints:

| Endpoint | Method | Purpose |
|---|---|---|
| `/health` | GET | Checks whether the API is running |
| `/transcribe` | POST | Receives an audio file and transcribes it |

The Whisper model is loaded when the application starts:

```python
model = WhisperModel(
    "tiny",
    device="cpu",
    compute_type="int8"
)
```

The `tiny` model is used because it is lightweight and suitable for this lab environment.

---

# Step 9 — Start the API

On the **Server VM**, activate the virtual environment:

```bash
source ~/whisper-api/venv/bin/activate
```

Go to the project directory:

```bash
cd ~/whisper-api
```

Start FastAPI:

```bash
uvicorn app:app --host 10.10.2.10 --port 8000
```

The server should now listen on:

```text
10.10.2.10:8000
```

Test the API locally on the Server:

```bash
curl http://10.10.2.10:8000/health
```

Expected output:

```json
{"status":"healthy"}
```

---

# Step 10 — Test Through IPSec

Now test whether the **Client VM** can access the private FastAPI service.

From the **Client VM**, run:

```bash
curl http://10.10.2.10:8000/health
```

Expected output:

```json
{"status":"healthy"}
```

This confirms that the Client can reach the private FastAPI endpoint through the IPSec-protected network.

The communication path is:

```text
Client
  |
  | HTTP Request
  |
  v
IPSec Tunnel
  |
  | Encrypted
  |
  v
Private Server
  |
  v
FastAPI
```

---

# Step 11 — Create Test Audio

Now create a sample audio file on the **Client VM**.

Install `espeak-ng`:

```bash
sudo apt install -y espeak-ng
```

Generate a test WAV file:

```bash
espeak-ng \
    -w sample.wav \
    "This is a secure voice model test."
```

Check that the file was created:

```bash
ls -lh sample.wav
```

---

# Step 12 — Send Audio to Whisper

From the **Client VM**, send the audio file to the private FastAPI endpoint:

```bash
curl -X POST \
    -F "file=@sample.wav" \
    http://10.10.2.10:8000/transcribe
```

The request travels through the IPSec tunnel to the private server.

Expected output:

```json
{
    "language": "en",
    "text": "This is a secure voice model test."
}
```

The complete inference flow is:

```text
sample.wav
    |
    v
Client VM
    |
    | IPSec / ESP
    |
    v
Server VM
    |
    v
FastAPI
    |
    v
Whisper
    |
    v
Transcription
```

---

# Step 13 — Verify Encryption

In this step, you will observe the network traffic generated by the IPSec tunnel.

First, identify the network interfaces:

```bash
ip -br addr
```

You may see something similar to:

```text
lo       UNKNOWN    127.0.0.1
eth0     UP         192.168.56.101
```

Use the appropriate transport interface in the next command.

For example:

```bash
sudo tcpdump -ni eth0 \
    'esp or udp port 500 or udp port 4500'
```

While `tcpdump` is running, send the audio again from another terminal:

```bash
curl -X POST \
    -F "file=@sample.wav" \
    http://10.10.2.10:8000/transcribe
```

You should observe IPSec-related traffic such as:

```text
ESP
UDP 500
UDP 4500
```

### What do they mean?

| Traffic | Purpose |
|---|---|
| UDP 500 | IKE negotiation |
| UDP 4500 | NAT Traversal (NAT-T) |
| ESP | Encrypted IPSec traffic |

The important observation is that the application data is carried through the IPSec protection mechanism instead of being directly exposed as ordinary network traffic between the transport endpoints.

---

# Verification

At the end of the lab, verify the following:

- [ ] Both Ubuntu VMs can communicate through the transport network.
- [ ] Private IP addresses are configured.
- [ ] strongSwan is installed on both VMs.
- [ ] IPSec tunnel is established.
- [ ] IKE SA is active.
- [ ] CHILD SA is active.
- [ ] XFRM state and policies exist.
- [ ] FastAPI server is running on the private server.
- [ ] `/health` endpoint responds successfully.
- [ ] Whisper successfully transcribes the test audio.
- [ ] Client can access the private inference endpoint.
- [ ] `tcpdump` shows IPSec-related traffic.
- [ ] Whisper is not exposed through a public IP.


# Conclusion

In this lab, you built a secure private voice inference system by combining **IPSec, strongSwan, FastAPI, and Whisper**.

First, you configured two Ubuntu VMs and established an IPSec tunnel between them. The tunnel used IKEv2 for negotiation and ESP to protect the traffic between the two endpoints. You then verified the IPSec connection using `ipsec statusall`, `ip xfrm state`, and `ip xfrm policy`.

Next, you deployed a Whisper speech-to-text model on the private server and exposed it through a FastAPI application. The client generated a sample audio file and sent it to the private FastAPI endpoint. The Whisper model processed the audio and returned the transcription result.

Finally, you used `tcpdump` to observe IPSec-related traffic such as ESP and UDP ports 500/4500. This helped verify that the communication was passing through the IPSec security layer.

The complete workflow can be summarized as:




The key lesson is that securing an ML application is not only about protecting the model or API. **The network path used to access the model is also an important part of the security architecture.**

By keeping the Whisper service on a private server and protecting communication with IPSec, sensitive voice data can be transported more securely between trusted systems. This architecture provides a basic foundation for designing more secure ML inference systems where model services do not need to be directly exposed to the public internet.