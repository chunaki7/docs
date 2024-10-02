# Docker VM GPU passthrough on Proxmox

### Setting up the VM in Proxmox

Add the GPU to your VM in Proxmox. Add > PCI Device and select the GPU from the drop down.

You'll want to edit the cpu flags for your VM. SSH into your Proxmox node and enter the following:

```bash
nano /etc/pve/qemu-server/107.conf
```

Next add the below flags to the cpu line.

```
cpu: host,hidden=1,flags=+pcid
```

### Ubuntu VM

Check that the GPU pops up with the below command

```bash
lshw -c video
```

Output

```
chunaki@docker-gpu:~$ lshw -c video
WARNING: you should run this program as super-user.
  *-display:0               
       description: VGA compatible controller
       product: bochs-drmdrmfb
       physical id: 2
       bus info: pci@0000:00:02.0
       logical name: /dev/fb0
       version: 02
       width: 32 bits
       clock: 33MHz
       capabilities: vga_controller bus_master rom fb
       configuration: depth=32 driver=bochs-drm latency=0 resolution=1024,768
       resources: irq:0 memory:f2000000-f2ffffff memory:fe4f4000-fe4f4fff memory:c0000-dffff
  *-display:1
       description: VGA compatible controller
       product: TU104GL [Quadro RTX 4000]
       vendor: NVIDIA Corporation
       physical id: 10
       bus info: pci@0000:00:10.0
       version: a1
       width: 64 bits
       clock: 33MHz
       capabilities: vga_controller bus_master cap_list rom
       configuration: driver=nouveau latency=0
       resources: irq:35 memory:fd000000-fdffffff memory:e0000000-efffffff memory:f0000000-f1ffffff ioport:e000(size=128) memory:c0000-dffff
WARNING: output may be incomplete or inaccurate, you should run this program as super-user.
```

#### Installing NVIDIA drivers

```bash
sudo apt update && apt upgrade -y
```

Search for latest nvidia drivers

```bash
sudo apt search nvidia-driver
```

Output:

```
...
xserver-xorg-video-nvidia-515-server/jammy-updates,jammy-security 515.86.01-0ubuntu0.22.04.2 amd64
  NVIDIA binary Xorg driver

xserver-xorg-video-nvidia-525/jammy-updates 525.85.05-0ubuntu0.22.04.1 amd64
  NVIDIA binary Xorg driver

xserver-xorg-video-nvidia-525-server/jammy-updates,jammy-security 525.85.12-0ubuntu0.22.04.1 amd64
  NVIDIA binary Xorg driver

```

The last one would be the latest one. At the time of writing, the latest working version is 535. Run the below commands referencing the latest version.

```bash
sudo apt install --no-install-recommends nvidia-headless-535-server
sudo apt install --no-install-recommends libnvidia-compute-535-server
sudo apt install --no-install-recommends nvidia-cuda-toolkit
sudo apt install --no-install-recommends nvidia-utils-535-server
sudo apt install --no-install-recommends libnvidia-encode-535-server
sudo apt install --no-install-recommends libnvidia-decode-535-server
```

Same as above but all in a single line instead.

```bash
sudo apt install --no-install-recommends nvidia-headless-535-server libnvidia-compute-535-server nvidia-cuda-toolkit nvidia-utils-535-server libnvidia-encode-535-server libnvidia-decode-535-server
```

#### Setting up Docker to recognise the GPU

```bash
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
```

```bash
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add - 
```

```bash
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
```

```bash
sudo apt update && sudo apt install -y nvidia-container-toolkit
```

```bash
sudo apt install nvidia-container-runtime
```

```bash
sudo apt install -y nvidia-docker2
```

We'll then need to edit the `daemon.json` file as below

```bash
sudo nano /etc/docker/daemon.json
```

Then remove whatever is in daemon.json and replace with the below.

```
{   
	"default-runtime": "nvidia",   
	"runtimes": {     
		"nvidia": {       
			"path": "/usr/bin/nvidia-container-runtime",       
			"runtimeArgs": []     
		}   
	} 
} 
```

#### Checking GPU functionality

Check that the VM can see the GPU

```bash
nvidia-smi
```

Check that the GPU is exposed to docker

```bash
docker run --rm --gpus all nvidia/cuda:12.1.0-base-ubuntu22.04 nvidia-smi
```

### Jellyfin GPU transcode

Settings > Dashboard > Playback

Check the formats based on the GPU you have referencing the nvidia [matrix](https://developer.nvidia.com/video-encode-and-decode-gpu-support-matrix-new).

You also need to pass the gpu through to your docker container. In ansible, I have the below:

```yaml
    device_requests:
      - device_ids: 0
        driver: nvidia
        capabilities:
          - gpu
```

> `device_ids` is the ID of the GPU that is obtained from `nvidia-smi -L`
>
> `capabilties` are spelled out on [nvidia’s repo](https://github.com/NVIDIA/nvidia-container-runtime#nvidia\_driver\_capabilities), but `all` doesn’t seem to work. {: .prompt-tip }

See Jellyfin's [documentation](https://jellyfin.org/docs/general/administration/hardware-acceleration/nvidia/#configure-on-linux-host) for more info.
