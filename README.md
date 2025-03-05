# ML-Based Helmet Detection

## WAGO Edge Computer and Hailo-8L™ M.2 Key B+M ET Module and Yolov5m

## Overview

This project deploys an AI inference service on a **WAGO Edge Computer 752-9813** equipped with the **Hailo-8L™ M.2 Key B+M ET Module**. The service supports both a **Logitech HD Pro Webcam C920** and a **Reolink E1 Pro RTSP camera** (or both) as image sources, based on your configuration. The solution uses Docker Compose (or a Portainer Stack) for containerized deployment and requires installation of specific Hailo drivers and tools.

---

## Hardware Requirements

- **WAGO Edge Computer 752-9813** 
  - ([**https://www.wago.com/global/edge-devices/edge-computer/p/752-9813**](https://www.wago.com/global/edge-devices/edge-computer/p/752-9813)**)**
  - Use a mSATA SSD flashed with the WAGO OS via the WAGO USB Installer. ([https://downloadcenter.wago.com/wago/software/details/lwq8xpd8d3np7r1rbxl](https://downloadcenter.wago.com/wago/software/details/lwq8xpd8d3np7r1rbxl))
- **Hailo-8L™ M.2 Key B+M ET Module**
  - Installed in the M.2 slot.
- **Camera Options***The following webcamhas been used:*
  - **Logitech HD Pro Webcam C920**

    - **Device:** `/dev/video0`
    - **Driver Info:**

      - **Driver Name:** `uvcvideo`
      - **Card Type:** `HD Pro Webcam C920`
      - **Bus Info:** `usb-0000:00:14.0-6`
      - **Driver Version:** `6.1.124`
      - **Capabilities:** `0x84a00001`


  - **Reolink E1 Pro RTSP Camera**

    - **Connection:** WiFi / LAN
    - **RTSP URL:** `rtsp://user:pw@ip:554/h264Preview_01_main`

---

## Installation and Setup

### 1. Prepare the Hardware

- **Flash the SSD:**
  - Download the WAGO OS image.
    - https://downloadcenter.wago.com/wago/software/details/lwq8xpd8d3np7r1rbxl
    - Follow the instrcutions and make sure
  - Use the WAGO USB Installer to flash the OS onto the mSATA SSD.
- **Assemble the System:**
  - Insert the flashed mSATA SSD into the WAGO Edge Computer.
  - Install the **Hailo-8L™ M.2 Key B+M ET Module** into the M.2 slot.
  - Connect the camera(s):
    - For the **Logitech Webcam**, plug it into a USB port.
    - For the **Reolink Camera**, ensure it is connected via WiFi.

### 2. Prepare the OS

- **GUI :**Make sure you have an installed on top of Debian 12 a GUI like LXDE (my case) or your Distro (e.g. Ubuntu 22.04) is shipped with one.

  ```bash
  sudo apt-get update -y && apt upgrade -y
  
  ```
  
  ```bash
  tasksel
  ```
  [x] LXDE
- **A Containerization:** Choose one of the following for deployment:

  - **Docker & Docker Compose**
    - Update/Install with:

      ```bash
      apt-get update
      apt-get install -y ca-certificates curl gnupg lsb-release
      install -m 0755 -d /etc/apt/keyrings
      curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
      chmod a+r /etc/apt/keyrings/docker.gpg
      echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg]    https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
      sudo apt-get update
      sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

      docker compose version
      ```

  **B Golden Image**

  - tbd.

---

### 3. Install Software Components

- **Create a account at https://hailo.ai/developer-zone/ **
- Go to https://hailo.ai/developer-zone/software-downloads/ and search for HailoRT & HailoRT – PCIe driver Ubuntu package (deb) --> download the package

- **Install Hailo driver:**
  ```bash
  https://hailo.ai/developer-zone/documentation/hailort-v4-20-0/?sp_referrer=install/install.html#ubuntu-installer-requirements
  ```

  ```bash
  sudo dpkg -i hailort-pcie-driver_4.20.0_all.deb
  ```
  ```bash
  reboot
  ```
    ```bash
  ls /dev/hailo
  ```


### 4. Verify HW

- **Optional:** Verify the camera feed (for the Logitech Webcam):

  ```bash
  sudo apt install -y v4l-utils

  v4l2-ctl --all --device /dev/video0
  ```

- **Optional:** Verify the camera feed (for the Logitech Webcam):

  ```bash
  sudo apt update
  sudo apt install ffmpeg -y
  ```
  ```bash
  ffmpeg -i 'rtsp://user:password@IP:554/h264Preview_01_main'
  ```
  ```bash
  ffmpeg -i 'rtsp://admin:Master1!@192.168.2.176:554/h264Preview_01_main'
  ```
  
- **Enable access to X Server (for the remote start of a inference window)**:

  ```bash
  sudo apt update
  sudo apt install xcb -y
  xhost +local:root
  ```


### 5. Configure the Deployment

- **Edit the ************************************************************************************************************************************************************************************************`docker-compose.yml`************************************************************************************************************************************************************************************************:**Adjust the entrypoint based on your camera source:

  - For **Webcam**:

    ```yaml
    entrypoint: ["./entrypoint.sh", "webcam"]
    ```

  - For **RTSP Camera**:

    ```yaml
    entrypoint: ["./entrypoint.sh"]
    ```
    
  DISPLAY & XDG_RUNTIME_DIR can be different on you host system, verify:
   
  ```bash
  echo $DISPLAY
  ```
  ":0"

  ```bash
  echo $XDG_RUNTIME_DIR
  ```

  "/run/user/0"
  
  If you want to transmit inference data, make sure to change:
  - MQTT_BROKER: "192.168.2.165"
  - MQTT_PORT: "1883"
  - MQTT_TOPIC: "inference/yolov5m-results"
  - CONFIDENCE_THRESHOLD: "0.42"
    
 Change if needed:
  - WEBCAM_INDEX: "0"
  - RTSP_URL: "rtsp://admin:Master1!@192.168.2.176:554/h264Preview_01_main"
- **Sample Docker Compose Configuration:**

  ```yaml

  services:
    hailo-ai:
      image: wagoalex/wago-hailo:yolov5m-helmet-v1.2.1
      container_name: wago-hailo-yolov5m-helmet
      privileged: true
      network_mode: host
      ipc: host
      #entrypoint: /bin/bash  # only for debugging
      #entrypoint: ["./entrypoint.sh","webcam"] # webcam usage
      entrypoint: ["./entrypoint.sh"] # rtsp usage
      environment:
        DISPLAY: ":0"
        XDG_RUNTIME_DIR: "/run/user/0"
        FRAME_WIDTH: "640"
        FRAME_HEIGHT: "640"
        CONFIDENCE_THRESHOLD: "0.42"
        MQTT_BROKER: "192.168.2.165"
        MQTT_PORT: "1883"
        MQTT_TOPIC: "inference/yolov5m-results"
        HEF_PATH: "/local/workspace/yolov5m-helmet.hef"
        WEBCAM_INDEX: "0"
        RTSP_URL: "rtsp://admin:Master1!@192.168.2.176:554/h264Preview_01_main"
      volumes:
        - /tmp/.X11-unix/:/tmp/.X11-unix/
        - /root/.Xauthority:/root/.Xauthority:ro
        - /dev:/dev
        - /lib/firmware:/lib/firmware
        - /docker/tests/:/local/workspace/share:rw
      devices:
        - /dev/dri:/dev/dri
      group_add:
        - 44
      tty: true
      stdin_open: true
  ```



- **Deploy the Application:**

  - With Docker Compose:

    ```bash
    docker compose up -d
    ```

- **Verify Deployment:**

  - Check container logs for errors:

    ```bash
    docker logs wago-hailo-yolov5m-helmet-compose
    ```

---

## Contact and Support

For any issues or further assistance, please contact:
**Email:** [iot@wago.com](mailto\:iot@wago.com)
**WAGO System Sales Germany**

---


---

*This documentation provides a comprehensive guide for first-time users to deploy the AI inference service on the WAGO Edge Computer platform with the Hailo-8L™ M.2 Key B+M ET Module.*
