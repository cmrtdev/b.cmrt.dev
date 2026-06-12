+++
date = '2026-06-02T23:42:46+02:00'
draft = true
title = 'How to bifurcate a PCI Express x16 slot (and lose your sanity in the process)'
+++

A few months ago I found myself in a bit of a predicament: my virtualized TrueNAS installation kept running into this weird issue where, at somewhat (but not really) random intervals, when the drives were a little bit overloaded, would give up on life, crash, and take Proxmox with it.

Of course, this happened only at the most convenient of times: when I was not home, in another country, in a hurry, etc.

My only solution, other than the good old "yank the power", was to ssh into the host and, well, yank the power, but different[^sysrq]:

```bash
echo 's' > /proc/sysrq-trigger
echo 'u' > /proc/sysrq-trigger
echo 's' > /proc/sysrq-trigger
echo 'b' > /proc/sysrq-trigger
```

and then wait, praying to The Homelab Gods that the server would reboot and at least my network and bastion host would come back online.

This was, for lack of a better word, _suboptimal_, but I couldn't really figure out what was happening, especially since the drives[^drives] were fine according to S.M.A.R.T. and everything else I could see, and I had PCI-E passthrough-ed them to TrueNAS to make sure TrueNAS had full control of them, so I (finally) started looking into it.

## PEBKAC

### A tale of four drives

I'll spare you the details, dear reader, but remember when, about two sentences ago I said this?

> [..] and I had PCI-E passthrough-ed them to TrueNAS to make sure TrueNAS had full control of them

Well, if your drives are attached to the motherboard, then you did NOT, in fact, passed them through: what I did was follow [this guide](https://pve.proxmox.com/wiki/Passthrough_Physical_Disk_to_Virtual_Machine_(VM)) from the Proxmox Wiki that, despite being called "Passthrough Physical Disk to Virtual Machine (VM)" does **not** pass the _hardware_ through, but - effectively[^kindasorta] creates a `virtio` "drive" representing the entire drive, and attaches it to the machine.

Crucially, however, **this does not pass the drive controller to the VM**, while Proxmox (via `virtio`) still acts as an "intermediary".

Now, a smarter person (a.k.a. someone with eyes and at least two brain cells) would've realized that something was off by looking at the "Storage > Disks" menu in TrueNAS, which clearly stated "Virtio device" (or something similar); and I did see that, but (see "missing brain cells" above) did not realize its implications.

### What have I done?

So, what had I actually done? Well, `virtio` is, by and large, a pretty amazing piece of software; but it's software nonetheless, and - worse - it's _generic_ software, which means it's not really capable of handling everything that the multitude of drive types it supports can do.

I don't know exactly what the issue is, I never figured that out, but what I do know is that - somewhere in the 6-to-8-weeks-ago realm, the crashes started getting more frequent, happening probably twice a week during nightly Proxmox backups[^hint], and I started to become more and more unnerved, so I decided to look into it a little more.

It took a relatively small amount of searching for the narrator standing above my shoulder to utter the very famous quote[^fuckedup]

> It was at this moment that he knew, he fucked up.

I pretty quickly came across [this post](https://forums.truenas.com/t/virtualized-truenas-core-crashes-during-proxmox-backup/34180) on TrueNAS forums, where OP was experiencing the same exact issue I was (YAY!) and a moderator helpfully pointed out that

> The setup you run is explicitly discouraged and will lead to data loss with a high probability. This has been discussed on this forum again and again.
>
> This is the only known to work configuration for a production system:
>
> <https://www.truenas.com/blog/yes-you-can-virtualize-freenas/>

_...nay?_

## Fixing the problem

### Just add another card

So how do I fix this? Very simple:

1. Get an HBA (Host Bus Adapter), a.k.a. a PCI Express card that does SATA
2. Plug it into a free PCIe port on my motherboard
3. Passthrough the HBA to TrueNAS
4. Profit!

So I go and find a nice, cheap (~$30 on eBay), LSI 9207-8i in `IT` mode (see [Choosing an HBA](#choosing-an-hba)) and I get ready to install it as soon as it gets here, when I realize... I don't have a PCIe slot for it.

My motherboard is a [MSI MAG B550 Tomahawk](https://www.msi.com/Motherboard/MAG-B550-TOMAHAWK), and it oh-so-helpfully comes with two PCIe x16 and two... PCIe x1 slots.

The x16 slots currently house a GPU (necessary, because the Ryzen 5800x in the server doesn't come with graphics. Thanks AMD!) and a 10G NIC (necessary, because internet go brr).

So, short of a rebuild of the server on a platform with more PCIe x16 slots (meaning AM5 or Intel, and have you seen the prices of RAM recently?!) bifurcation is needed: the NIC, a Mellanox ConnectX-3, only needs an 8x slot, and so does the HBA. I can split the x16 slot into two x8 and plug the cards into it. Not a big deal, should work, right?

Yes, except that the last time I tried to do passthrough with the NIC it **froze the entire machine**, because the PCIe slot was in the same [IOMMU group](https://en.wikipedia.org/wiki/Input%E2%80%93output_memory_management_unit) as - oh I don't know - THE USB CONTROLLER.

I briefly consider sawing off the back of one of the x1 slots and plugging the HBA or NIC in it, leaving the other card in the x16, but math is not on my side:

* My ConnectX-3 is dual 10G PCIe 3.0, meaning 8GT/s (4Gbps) per lane; I need at least 3 lanes (`10 Gbps / 4Gbps = 2.5`[^latex]) to make it do 20Gbps, twice that for 40.
* The HBA will have to support four 6Gbps SATA drives, `6*4=24`, three lanes again[^spinningrust].

PCIe 3.0 x1 won't do for either of them.

Some thinking and research[^rtfm] later, I have a plan:

* Put the GPU in `PCI_E3`: I'm not passing it through to any VM anyway
* Put a bifurcation adapter in `PCI_E1` and plug both cards in
* Enable x8x8 bifurcation in the BIOS
* Passthrough the HBA to TrueNAS
* Profit!

So off I go to AliExpress to find a `PCIe bifurcation riser`, order it, and wait.

### Where does the square peg go? In the square hole!

As I'm impatiently waiting for everything to get delivered I make a sudden realization:

* I bought a 90 degrees bifurcation adapter
* That plugs in horizontally, and the cards go vertically
* My case is a [Fractal Define R5](https://www.fractal-design.com/products/cases/define/define-r5/)
* It does not have vertical holes for PCIe cards
* And even if it did, I can't reach them

The solution: riser cables! I can go from the two slots to each PCIe!

Actually, I can buy a single x16 riser cable and go from the motherboard to the bifurcation riser, then slot the cards one above the other.

I see absolutely **no way** this plan can go wrong!

### Physics, you heartless bitch



## Appendix

### Choosing an HBA

There are a million[^probably] people, like the fine folks at [ServeTheHome](https://www.servethehome.com/), that can tell you about HBAs, their firmware, weird combinations, etc. I won't do that.

Suffice to say that I had a choice between the [`9207-8i`](https://docs.broadcom.com/doc/12353331) I bought and a more expensive `9217-8i` or even something newer, but I read good things online about the `9207-8i`, including the fact that it came in `IT` (Initiator Target) mode by default instead of `IR` (Integrated RAID), which is more datacenter-y.

To use an HBA in `IR` mode with TrueNAS I'd have had to reflash it to `IT` mode, effectively turning it into a `9207-8i`. So, more money and setup effort. Have I not suffered enough?

[^sysrq]: in case you don't know, this how you 'press' the [Magic System Request (SysRq) keys](https://docs.kernel.org/admin-guide/sysrq.html) `Sync`, `Unmount`, `Sync`, and `reBoot` when you don't have a keyboard plugged into the machine, but happen to have a terminal.
[^drives]: 4x Seagate Exos X16, model `ST16000NM001G`
[^kindasorta]: this may be the wrong explanation, but it works for our purposes.
[^hint]: <https://en.wikipedia.org/wiki/Foreshadowing>
[^fuckedup]: <https://knowyourmeme.com/memes/it-was-at-this-moment-he-knew-he-fucked-up>
[^probably]: Probably more, or maybe less.
[^rtfm]: I figured out that PCI_E1 (topmost slot) is directly connected to the processor, while PCI_E2, PCI_E3, and PCI_E4 (bottom slots) go through the chipset, which puts them into the same IOMMU group as too many other devices. This is only **hinted at** on PAGE SIXTEEN of [the manual](https://download-2.msi.com/archive/mnu_exe/mb/MAGB550TOMAHAWK.pdf), AND you need know what you're looking for! Dammit MSI, do better, this was painful!
[^spinningrust]: and yes, I know that these are four "spinning rust" drives and they will probably never see anywhere near 6Gbps, lest they take off like a frenzied helicopter, but I'm not risking it. I want the full bandwidth available.
[^latex]: I'm not adding LaTeX support to Hugo just to show multiplication, sorry.