# Microelectronics US 2026 Thistle Workshop Attendee Guide

<p align="center">
  <img src="img/thistle-ai-model-deploy.svg" alt="Thistle AI model OTA deploy
  data flow: TRH on a laptop encrypts and signs model.pt, releases the OTA
  bundle to Thistle Control Center, TUC on a Raspberry Pi fetches and verifies
  it, and decrypts the model for the AI app at inference time." width="960">
</p>

In this workshop, you will learn how to:

* Use the TRH ([Thistle Release
  Helper](https://docs.thistle.tech/binaries)) command-line tool to encrypt and
  digitally sign a pre-trained AI model, and publish it to [Thistle Control
  Center](https://app.thistle.tech) (TCC) for Over-The-Air (OTA) deployment to a
  Raspberry Pi 4 device.
* Use TUC ([Thistle Update Client](https://docs.thistle.tech/binaries)) on the
  Raspberry Pi 4 to securely fetch the OTA bundle, verify the AI model's digital
  signature, and decrypt the model when it is loaded for inference.
* Use the Infineon OPTIGA™ Trust M secure element to manage the public key that
  verifies both the OTA bundle and the AI model.

## Step 1: Set up the workshop laptop

1. Connect the laptop to the workshop Wi-Fi hotspot (SSID: `Thistle-workshop`).

2. Confirm that the workshop directory `/home/thistle/meus2026-workshop/`
   exists. If it does not, clone this repository:

   ```bash
   # install these programs if they are not already installed
   sudo apt install curl minicom vim git
   # change to the home directory (/home/thistle)
   cd
   # clone this repo
   git clone https://github.com/thistletech/meus2026-workshop-thistle-guide.git
   ```


## Step 2: Run the demo application without an AI model

1. A USB-to-TTL serial cable has already been connected between the laptop and
   the Raspberry Pi 4. Open a terminal on the laptop and attach to the serial
   console:

   ```bash
   minicom -b 115200 -D /dev/ttyUSB0
   ```

2. Make sure the Raspberry Pi is powered on. At the login prompt, log in with
   username `thistle` and password `raspberry`.

   - Find the Raspberry Pi's IP address and note it down:

     ```bash
     thistle@thistle-meus26:~ $ ip addr show wlan0
     ```

   - Run the demo application without a model:

     ```bash
     # Clean up first
     thistle@thistle-meus26:~ $ rm -rf ~/.thistle-meus26-demo/ota
     thistle@thistle-meus26:~ $ sudo umount ~/.thistle-meus26-demo/ramdisk

     thistle@thistle-meus26:~ $ . ~/.thistle-meus26-demo/venv/bin/activate
     thistle@thistle-meus26:~ $ cd ~/meus2026-workshop
     thistle@thistle-meus26:~/meus2026-workshop $ . demo/vars.env
     thistle@thistle-meus26:~/meus2026-workshop $ python demo/app/demo.py
     ```

3. On the laptop, browse to `http://[RPi IP ADDRESS]:5000/`. You should see a
   video clip titled "Live Streaming (AI model isn't applied)".

## Step 3: Securely deploy the AI model and run the demo

### 1. Sign up on TCC

On the laptop, open a browser in privacy/incognito mode and sign up at [Thistle
Control Center](https://app.thistle.tech). Once logged in, create a new project
and give it a name (for example, "MEUS 2026 Workshop"). Then visit **Settings >
Access > Project** and copy the **Project Access Token** to the clipboard.

### 2. Publish the encrypted, signed AI model and generate a device configuration

In a terminal on the laptop:

```bash
# Change to the repo directory
cd ~/meus2026-workshop-thistle-guide/
# Paste your Thistle "Project Access Token", then press Ctrl+D
export THISTLE_TOKEN=$(cat)

# Activate the hermit environment
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

# Release the OTA bundle to TCC
trh --signing-method="remote" release --name="AI model"

# Generate the TUC config
trh --signing-method="remote" gen-device-config \
  --device-name="demo-device" \
  --enrollment-type="pre-enroll" \
  --persist="${DEMO_PERSIST_DIR}" \
  --config-path="./tuc-config.json"
```

Open `tuc-config.json` and add `"single_check": true` to the end of the JSON
object. For example:

```json
{
    "name": "demo-device",
    "persistent_directory": "/home/thistle/.thistle-meus26-demo",
    ...,
    "single_check": true
}
```

Copy `tuc-config.json` to the Raspberry Pi:

```bash
scp tuc-config.json thistle@[RPi IP ADDRESS]:~/meus2026-workshop/demo/resources/
```

### 3. Run the demo application on the Raspberry Pi

On the serial terminal connected to the Raspberry Pi:

```bash
thistle@thistle-meus26:~ $ cd ~/meus2026-workshop/
thistle@thistle-meus26:~/meus2026-workshop $ ./demo/thistle-demo.sh run
```

You should see a verification-failure error — this is expected. Press `Ctrl-C`
to stop the application, then edit `demo/app/thistle_secure_loader.py` and fill
in the `verify_model` and `decrypt_model` functions with the
signature-verification and decryption snippets.

Run the demo application again:

```bash
thistle@thistle-meus26:~/meus2026-workshop $ ./demo/thistle-demo.sh run
```

On the laptop, browse to `http://[RPi IP ADDRESS]:5000/` to confirm that the
decrypted model is securely loaded for inference.

When you are done, press `Ctrl-C` on the serial terminal to stop the demo
application.

## Step 4: Run the demo using the OPTIGA™ Trust M for key management

1. In a browser on the laptop, open your TCC project, go to
   **Settings > Access > Releases**, and copy the
   **OTA Public Verification Key** to the clipboard.

2. On the serial terminal connected to the Raspberry Pi, provision the OTA
   public key to the Trust M:

   ```bash
   thistle@thistle-meus26:~ $ cd ~/meus2026-workshop/
   # paste [OTA PUBKEY] from clipboard
   thistle@thistle-meus26:~/meus2026-workshop $ echo -n "[OTA PUBKEY]" > ota-pubkey.txt

   # write the public key to Trust M
   thistle@thistle-meus26:~/meus2026-workshop $ ./demo/provision-trustm.sh ota-pubkey.txt
   ```

3. On the Raspberry Pi, edit
   `~/meus2026-workshop/demo/resources/tuc-config.json` and, in the
   `public_keys` array, replace `"ecdsa:..."` with `"trustm:/dev/i2c-1"`. The
   result should look like:

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

4. Run the demo application on the Raspberry Pi to use the Trust M for OTA
   bundle and AI model verification:

   ```bash
   thistle@thistle-meus26:~ $ cd ~/meus2026-workshop/
   thistle@thistle-meus26:~/meus2026-workshop $ ./demo/thistle-demo.sh run
   ```

5. On the laptop, browse to `http://[RPi IP ADDRESS]:5000/` to confirm that the
   decrypted model is securely loaded for inference.

## Step 5: Clean up

When you are done, on the laptop PC:

- On the serial terminal connected to the Raspberry Pi, poweroff the device

  ```bash
  thistle@thistle-meus26:~/meus2026-workshop $ sudo poweroff
  # press Ctrl-A, X to exit from minicom
  ```

- Close the browser (and clear browser cache if needed)

- Remove the cloned repo

  ``bash
  rm -rf ~/meus2026-workshop-thistle-guide
  ```