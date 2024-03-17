# Launching Livestream for Isaac Sim (on A100)

## Ways to Launch Livestream Isaac Sim from a Remote Server

There are 3 ways to stream images from a remote server:


1. **Omniverse Streaming Client** - the most stable and recommended, but not suitable for all models
2. **WebRTC Browser Client**
3. **WebSocket Browser Client** - partially removed in the 2023 version and will be completely removed in 2024


:::info
If you opened this guide, most likely the [main instruction](https://docs.omniverse.nvidia.com/isaacsim/latest/installation/manual_livestream_clients.html) on the Nvidia website did not help you, so here I will tell you what I managed to learn by trial and error 
:::

## Minimal Graphics Card Requirements


:::warning
GPU **must** support **RTX** technology ([brief list of supported GPUs](https://en.wikipedia.org/wiki/Nvidia_RTX))
:::

## Highly Desirable Graphics Card Requirement

For streaming through **Omniverse Streaming Client** and **WebRTC**, GPU must support hardware video encoding and decoding - **NVENC**, you can familiarize yourself with the list of supported devices [here](https://developer.nvidia.com/video-encode-and-decode-gpu-support-matrix-new)


:::success
If your card supports NVENC, you can safely start streaming through **Omniverse Streaming Client** [instructions](https://docs.omniverse.nvidia.com/streaming-client/latest/user-manual.html) or through **WebRTC Streaming Client** [instructions](https://docs.omniverse.nvidia.com/extensions/latest/ext_livestream/webrtc.html), just don't forget to open the necessary ports 
:::

## Non-obvious Mini List of Server Card Support

**Supported:** T4, A10

**Partially supported:** A100

**Not supported:** V100, P100, H100


> Tested on A100 and T4 myself


:::info
Nvidia's hardware recommendations for Isaac Sim: [link](https://docs.omniverse.nvidia.com/isaacsim/latest/installation/requirements.html)*[1](https://docs.omniverse.nvidia.com/isaacsim/latest/installation/requirements.html), [link](https://docs.omniverse.nvidia.com/deployment/latest/ra-non-virtualized.html#nvidia-rtx-gpu-recommendations-for-professional-workstation-users)*[2](https://docs.omniverse.nvidia.com/deployment/latest/ra-non-virtualized.html#nvidia-rtx-gpu-recommendations-for-professional-workstation-users)
:::

## What to Do If NVENC Is Not Supported, But RTX Support Exists? (A100)

In this case, there is only one way out - to start streaming through **Websocket** using **H.264** decoding standard



1. Make sure that the supported drivers for the GPU are installed, or install them yourself

   You can find out the driver version using `nvidia-smi` 


:::success
**Recommended drivers** 

Linux: 525.85.05, 525.85.12 (Grid/vGPU) 

Windows: 528.24, 528.33 (Grid/vGPU)
:::


:::warning
**Unsupported drivers** 

Linux: all before 510.73.05, from 515.0 to 515.17

Windows: all before 473.47, from 495.0 to 512.59, from 525 to 526.91
:::

**Installation:**

```bash
sudo apt-get update
sudo apt install build-essential -y
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/525.85.05/NVIDIA-Linux-x86_64-525.85.05.run
chmod +x NVIDIA-Linux-x86_64-525.85.05.run
sudo ./NVIDIA-Linux-x86_64-525.85.05.run
```

1. Install Isaac Sim 2022.2.1 - as this is the latest version of the simulator on which streaming was successful

2. Make sure that TCP ports 8899 and 8211 are open. To do this, you need to start streaming, using netstat -lntu or ss -lntu

3. If the ports are closed, you can try to open them with the following commands:

 **For standard Ubuntu firewall:**

```bash
sudo ufw allow port_name 
```
**For other cases:**

```bash
iptables -A INPUT -p tcp --dport port_name -j ACCEPT 
sudo invoke-rc.d iptables-persistent save
```

If it didn't help, most likely there are additional access settings on the remote server, so ask the sysadmin to open the ports


5. Check the connection to the port: `telnet localhost port_number`



6. If you use a Docker container, don't forget to specify --network=host or forward **8899** and **8211** ports, run the docker container:

   ```bash
   docker run --name isaac-sim --entrypoint bash -it --gpus device=1 
   -e "ACCEPT_EULA=Y" --rm --network=host     -e "PRIVACY_CONSENT=Y"     
   -v ~/docker/isaac-sim/cache/kit:/isaac-sim/kit/cache:rw     
   -v ~/docker/isaac-sim/cache/ov:/root/.cache/ov:rw     
   -v ~/docker/isaac-sim/cache/pip:/root/.cache/pip:rw     
   -v ~/docker/isaac-sim/cache/glcache:/root/.cache/nvidia/GLCache:rw     
   -v ~/docker/isaac-sim/cache/computecache:/root/.nv/ComputeCache:rw     
   -v ~/docker/isaac-sim/logs:/root/.nvidia-omniverse/logs:rw     
   -v ~/docker/isaac-sim/data:/root/.local/share/ov/data:rw     
   -v ~/docker/isaac-sim/documents:/root/Documents:rw  
   -v ~/docker/isaac-sim/extension_examples:/isaac-sim/extension_examples:rw 
   -v ~/docker/isaac-sim/standalone_examples:/isaac-sim/standalone_examples:rw   
   nvcr.io/nvidia/isaac-sim:2022.2.1
   ```
7. **TO-DO: Add changing default ports** 
8. Launch streaming

   **Linux:**

```bash
./isaac-sim.headless.websocket.h264.sh
```

**Windows**: 

```javascript
./isaac-sim.headless.websocket.h264.bat
```

**Docker container:** 

```javascript
./runheadless.websocket.h264.sh
```


 8. To run as a standalone application, add the following code:

    ```bash
    from omni.isaac.kit import SimulationApp
    
    CONFIG = {
        "width": 1280,
        "height": 720,
        "window_width": 1920,
        "window_height": 1080,
        "headless": True,
        "renderer": "RayTracedLighting",
        "display_options": 3286,  # Set display options to show default grid
    }
    
    # it is important to run before the imports of the other libraries, otherwise you will get an error
    simulation_app = SimulationApp(launch_config=CONFIG)
    
    simulation_app.set_setting("/app/window/drawMouse", True)
    simulation_app.set_setting("/app/livestream/proto", "ws")
    simulation_app.set_setting("/app/livestream/websocket/framerate_limit", 120)
    simulation_app.set_setting("app/livestream/websocket/encoder_selection", 'OPENH264')
    simulation_app.set_setting("/ngx/enabled", False)
    
    
    from omni.isaac.core.utils.extensions import enable_extension
    
    enable_extension("omni.services.streamclient.websocket")
    
    #
    # YOUR LIBRARIES IMPORT AND CODE HERE
    #
    
    
    while simulation_app.is_running() and not simulation_app.is_exiting():
    
        #
        # YOUR CODE HERE
        #
        
        simulation_app.update()
    simulation_app.close()
    
    ```
 9. Aftre all run app: `./python.sh ./path/to/app/app_name.py` and connect via browser to `http://localhost:8211/streaming/client/` preferably via chrome
10. **TO-DO: add about ssh tunneling for vs code**

