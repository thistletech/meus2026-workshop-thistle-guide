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



### Step 3: Sign up on Thistle Control Center (TCC)

Sign up in Thistle Control Center (https://app.thistle.tech). Once logged in,
create a new project, and name it "MEUS 2026 Workshop" - you may also choose
another project name you like.

## Run demo app without a model

On RPi

. ~/.thistle-meus26-demo/venv/bin/activate
cd ~/meus2026-workshop
. demo/vars.env
python demo/app/demo.py

Browse to http://<rpi-ip-addr>:5000/

## Run demo app with end-to-end encrypted model

1. Sign up with Thistle
2. Create a new project
3. Copy project acccess token

# At the repo's directory root
export THISTLE_TOKEN=$(cat)
# Paste your Thistle "Project Access Token", then press Ctrl+d

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

trh --signing-method="remote" gen-device-config --device-name="demo-device" --enrollment-type="pre-enroll" --persist="${DEMO_PERSIST_DIR}"  --config-path="./tuc-config.json"

Add `"single_check": true` to the end of the JSON file

scp tuc-config thistle@<rpi-ip-address>:~/meus2026-workshop/demo/resources/

### With Trust M

Copy public key to clipboard

# No trailing whitespace
echo -n "ecdsa:blahblahblah" > ota-pubkey.txt

./demo/provision-trustm.sh ota-pubkey.txt

In tuc-config.json, in public_keys array, replace "ecdsa:..." with "trustm:/dev/i2c-1".

./demo/app/demo.py run