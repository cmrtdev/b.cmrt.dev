+++
date = '2026-01-31T16:14:29+01:00'
title = 'Immich ML Hardware Acceleration with OpenVINO in a Proxmox LXC container'
+++

Also known as: [I am stupid](https://www.youtube.com/watch?v=aS4Me48wayM).

## What?

(This is hopefully going to be a short one)

I recently installed [Immich](https://immich.app/) as I'm trying to get away from backing up my pictures on [someone else's computer](https://blog.codinghorror.com/the-cloud-is-just-someone-elses-computer/), or on NextCloud (awesome tool, horrible auto-sync behavior in the Android app, no fancy features for photo hosting).

I chose to run it in an LXC container because it allows me to let it use [my GPU](https://www.intel.com/content/www/us/en/products/sku/227959/intel-arc-a380-graphics/specifications.html) without having to do PCIe Passthrough, which reserves the card for that machine[^because]. I'm also running the default Docker Compose setup (yeah, yeah, container in a container... I don't care).

The [official instructions](https://docs.immich.app/features/ml-hardware-acceleration/) from the Immich team are on point: I uncommented all the things, chose the correct backend, did all the things... But still, every time I checked, the only available ML provider was `CPUExecutionProvider`.

## Why?

I actually delayed starting using Immich until I figured this out, but after many reinstalls, in multiple separate occasions, installing the `intel-opencl-icd` in all the affected machines[^unstable], and still never seeing the `OpenVINOExecutionProvider` even being available I had all but given up.

So I did what any sane person would do, and removed all the configs[^uhhuh] and started from scratch. And that's when I (finally?) spotted this comment in Immich's [docker-compose.yml](https://github.com/immich-app/immich/blob/main/docker/docker-compose.yml):

```yaml
  immich-machine-learning:
    container_name: immich_machine_learning
    # For hardware acceleration, add one of -[armnn, cuda, rocm, openvino, rknn] to the image tag.
    # Example tag: ${IMMICH_VERSION:-release}-cuda
    image: ghcr.io/immich-app/immich-machine-learning:${IMMICH_VERSION:-release}
```

What?

> _For hardware acceleration, **add one of -[...] to the image tag**_

I tried it, it worked instantly[^tbf].

At which point, after writing a whole paragraph about "why is this not in the official instructions?" I checked them for what feels like the 75th time[^ydwbroti], and saw it:

> 3. Still in `immich-machine-learning`, add one of `-[armnn, cuda, rocm, openvino, rknn]` to the `image` section's tag at the end of the line.

While I don't agree with this being `step 3`, when `step 2` involves uncommenting and changing lines that are below it in the `docker-compose.yml` file, or the fact that they could name the default image `cpu`, `noaccel`, or something that makes it more obvious that it needs to be changed, I am definitely [nitpicking](https://www.merriam-webster.com/slang/copium) here.

The underlying point, which I also made at the very start of this, remains: [I am stupid](https://www.youtube.com/watch?v=aS4Me48wayM). But hey, at least my Immich ML works now.

[^because]: I also run Jellyfin with hardware transcoding, and I like my server to have a video out, which the Ryzen 5800X does not come with.
[^unstable]: Coincidentally: did you know `intel-opencl-icd` is only available in Debian Unstable? I didn't, that was fun to figure out!
[^uhhuh]: I definitely did not accidentally overwrite my `.env` file while reinstalling Immich for the umpteenth time, your honor.
[^tbf]: To be fair, in the process I had also fixed the device passthrough - which now Proxmox allows you to do directly from the UI - and installed _All The Drivers™_.
[^ydwbroti]: There's not much worse in life than being wrong on the internet.
