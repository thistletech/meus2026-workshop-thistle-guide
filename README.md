# Microelectronics US 2026 Thistle Workshop Attendee Guide

<p align="center">
  <img src="img/thistle-ai-model-deploy.svg" alt="Thistle AI model OTA deploy
  data flow: TRH on a laptop encrypts and signs model.pt, releases the OTA
  bundle to Thistle Control Center, TUC on a Raspberry Pi fetches and verifies
  it, and decrypts the model for the AI app at inference time." width="960">
</p>

In this workshop, we will learn how to

* Use TRH ([Thistle Release
  Helper](https://docs.thistle.tech/binaries)) command-line tool to publish an
  encrypted and digitally signed pre-trained AI model to [Thistle Control
  Center](https://app.thistle.tech), for OTA (Over-The-Air) deploying the model
  to a Rasbperry Pi 4 device.
* Use TUC ([Thistle Update Client](https://docs.thistle.tech/binaries)) to
  securely OTA deploy the published AI model on the Raspberry Pi 4, and verify
  the model's digital signature as well as decrypts the AI model when the AI
  model needs to be loaded for inference.
* Use the Infineon OPTIGA™ Trust M secure element to manage the public
  verification key for verifying the OTA update and the AI model.

## Step 1: Set up workshop laptop

1. Make sure the laptop is connected to the workshop Wi-Fi hotspot (SSID:
   `Thistle-workshop`).
   
2. Check that the workshop directory `/home/thistle/meus2026-workshop/` is
   present. If it's not there, clone this repository

   ```bash
   # change to home directory (/home/thistle)
   cd
   # clone this repo
   git clone https://github.com/thistletech/meus2026-workshop-thistle-guide.git
   ```


## Step 2: Run demo application without using an AI model

1. Use a USB to TTL serial cable to connect the laptop PC and the Raspberry Pi 4
   - this should have already been set up for you. In a terminal window on the PC

   ```bash
   minicom -b 115200 -D /dev/ttyUSB0
   ```

2. Make sure the Raspberry Pi is powered on. On the login prompt, use username
   `thistle` and password `raspberry` to login to the Raspberry Pi.

   - Find out the IP address of the Rasbperry Pi, and take note on it.

     ```bash
     thistle@thistle-meus26:~ $ ip addr show wlan0
     ```

   - Run the demo AI application without a model

     ```bash
     # Clean up first
     thistle@thistle-meus26:~ $ rm -rf ~/.thistle-meus26-demo/ota
     thistle@thistle-meus26:~ $ sudo umount ~/.thistle-meus26-demo/ramdisk

     thistle@thistle-meus26:~ $ . ~/.thistle-meus26-demo/venv/bin/activate
     thistle@thistle-meus26:~ $ cd ~/meus2026-workshop
     thistle@thistle-meus26:~ $ . demo/vars.env
     thistle@thistle-meus26:~ $ python demo/app/demo.py
     ```

3. On laptop PC, use a browser to view `http://[RPi IP ADDRESS]:5000/`. You
   should see a video clip with title "Live Streaming (AI model isn't applied)".

### Step 3: Securely deploy AI model and run demo application

### 1. Sign up on TCC

Open a browser on laptop PC.  Sign up in Thistle Control Center
(https://app.thistle.tech). Once logged in, create a new project, and give it a
name, e.g., "MEUS 2026 Workshop". Visit "Project Settings > Access > Project",
and copy the "Project Access Token" to clipboard.

### 2. Use TRH to publish encrypted and signed pre-trained AI model and create new device configuration

In a terminal window on laptop PC

```bash
# Change to the repo's directory
cd ~/meus2026-workshop/
# Paste your Thistle "Project Access Token", then press Ctrl+d
export THISTLE_TOKEN=$(cat)

# Active hermit environment
. bin/activate-hermit

export DEMO_PERSIST_DIR="/home/thistle/.thistle-meus26-demo"

# Initialize the TCC project
trh --signing-method="remote" init

# Fetch the AI model and place it in the OTA staging directory
mkdir -p ota/
curl -L -o ota/model.pt https://downloads.thistle.tech/thistle-ifx-rpi4-demo/model.pt

# Prepare the OTA release (signed and encrypted)
trh --signing-method="remote" prepare \
  --target="ota" \
  --file-base-path="${DEMO_PERSIST_DIR}/ota" \
  --encrypt-ai-model \
  --zip-target \
  --encrypt-ota \
  --sign-ai-model

# Release OTA bundle to TCC
trh --signing-method="remote" release --name="AI model"

# Generate tuc config
trh --signing-method="remote" gen-device-config \
  --device-name="demo-device" \
  --enrollment-type="pre-enroll" \
  --persist="${DEMO_PERSIST_DIR}" \
  --config-path="./tuc-config.json"
```

Open `tuc-config.json`, and add `"single_check": true` to the end of the JSON block. Example

```json
{
    "name": "demo-device",
    "persistent_directory": "/home/thistle/.thistle-meus26-demo",
    ...,
    "single_check": true
}
```

Copy `tuc-config.json` to the Raspberry Pi

```bash
scp tuc-config thistle@[RPi IP ADDRESS]:~/meus2026-workshop/demo/resources/
```

### 3. Run demo application on Raspberry Pi

On Raspbery Pi serial port terminal

```bash
cd ~/meus2026-workshop/
./demo/thistle-demo.sh run
```

You should see an error indicating verification failure. This is expected.
`Crtl-C` to stop the application. Edit file `demo/app/thistle_secure_loader.py`
to add code snippets in functions `verify_model` and `decrypt_model` for AI
model signature verification and decryption.

Run the demo application again.

```bash
./demo/thistle-demo.sh run
```

On laptop PC, browse to `http://[RPi IP ADDRESS]:5000/` to confirm that the
decrypted model is securely loaded for inference.

On the serial terminal connected to the Raspberry Pi, `Ctrl-C` to stop the demo
application when you are done.

## Step 4: Run demo application, using the OPTIGA™ Trust M for key management

1. On a browser on laptop PC, in your TCC project, go to "Settings > Access >
   Releases" and copy the "OTA Public Verification Key" to clipboard.

2. On the serial terminal connected to the Rasbperry Pi, provision the OTA
   public key to the Trust M.

   ```bash
   cd ~/meus2026-workshop/
   # paste [OTA PUBKEY] from clipboard
   echo -n "[OTA PUBKEY]" > ota-pubkey.txt

   # write public key to Trust M
   ./demo/provision-trustm.sh ota-pubkey.txt
   ```

3. On Raspberry Pi, edit `~/meus2026-workshop/demo/resources/tuc-config.json`,
   in the `public_keys` array, replace `"ecdsa:..."` with `"trustm:/dev/i2c-1"`.
   The result looks like

   ```json
   {
       "name": "demo-device",
       "persistent_directory": "/home/thistle/.thistle-meus26-demo",
       "public_keys": [ "trustm:/dev/i2c-1" ],
       "device_id": ...,
       ...,
       "single_check": true
   }
   ```

4. Run the demo application on Raspberry Pi to use the Trust M for OTA bundle
   and AI model verification

   ```bash
   cd ~/meus2026-workshop/
   ./demo/app/demo.py run
   ```

5. On laptop PC, browse to `http://[RPi IP ADDRESS]:5000/` to confirm that the
   decrypted model is securely loaded for inference.
