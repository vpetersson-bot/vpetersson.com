---
slug: state-of-the-commercial-raspberry-pi-ecosystem
title: "The State of Commercial Raspberry Pi Security: I Scanned Every Paid-For OS in Raspberry Pi Imager"
date: '2026-11-26T09:00:00Z'
description: "I downloaded every image in the 'Freemium and paid-for OS' section of Raspberry Pi Imager, generated an SBOM for each, and ran them through OSV. End-of-life Debian, Python 2.7, stale Chromium, a default password, shared SSH host keys, and thousands of already-fixed CVEs -- on devices that live inside business networks."
tags:
- security
- raspberry-pi
- sbom
- cra
- iot
- digital-signage
---

A while back, in [The Last 20%](/2026/08/21/the-last-20-percent/), I mentioned in passing that the digital signage industry's security posture is "pretty terrible." That was an aside, thrown in next to a point about bad products and high subscription fees. This post is me actually checking whether I was right.

Here is the setup. The Raspberry Pi Imager, the official tool for flashing SD cards, has a section called **Freemium and paid-for OS**. These are images from companies selling a product or a service on top of the Pi: digital signage players, a thin client, a smart-home hub, a 3D printer controller, a commercial Android build. These are not weekend projects. They end up in lobbies, meeting rooms, shop windows, factory floors and hospital corridors, almost always on the corporate network, usually on a VLAN nobody has looked at since install day, and almost never in anyone's asset inventory.

So I did the obvious thing. I downloaded each vendor's image, generated an SBOM, and checked it for known vulnerabilities. The results are worse than my offhand "pretty terrible" suggested.

A note before the numbers: hobbyist builds get a pass here, and that is deliberate. Nobody expects a retro-gaming setup or a kid's weather station to ship with a vulnerability management program, and holding a maker flashing an image for a weekend to the same bar as a company selling a product would be unfair. The person tinkering has made no promises to anyone. A vendor charging for a signage player has, whether they admit it or not, and starting in December 2027 the EU will make that obligation explicit. So I scanned only the things sold as products. That is why I limited this to the paid-for category and left every other section of Imager alone.

## How I did it

1. I pulled the Raspberry Pi Imager OS list (`os_list_imagingutility_v4.json`) on 20 November 2026 and took every entry under **Freemium and paid-for OS**.
2. For each vendor I grabbed the newest image that supports a Pi 4 or Pi 5. One scan per vendor, the most current build.
3. I decompressed each image, attached it as a read-only loop device, and mounted every partition. Where the root filesystem was buried inside Ubuntu Core snaps, balenaOS container layers, an LVM volume or a squashfs payload, I mounted that too.
4. I recorded the OS release, kernel, package sources and update mechanism, then generated an SBOM with [Syft](https://github.com/anchore/syft) 1.57.2 and scanned it with [OSV-Scanner](https://github.com/google/osv-scanner) 2.7.1.

A few things to keep in mind when reading the table:

- **Counts are unique advisories** after OSV groups aliases. A CVE that also has a DSA, GHSA or PYSEC ID is counted once.
- **"Fix available"** means OSV knows a patched version exists in that ecosystem. In other words, the problem is already solved upstream and the vendor simply hasn't shipped it.
- **The kernel is counted separately.** Since the Linux kernel became its own CNA it issues thousands of CVEs a year, and matching Pi kernels against Debian's `linux` advisories is mostly noise. I report it but keep it out of the headline numbers.
- **Raspbian has no OSV ecosystem of its own**, so I mapped Raspbian packages to the matching Debian release and used source package names, which is how Debian keys its advisories.
- **This is the out-of-the-box state.** Some products update themselves after first boot, so a device in the field may be better (or worse) than its image. But the image is what the vendor chose to ship, and it is where every new device starts.

## The results

| Product (image tested)                         | Base OS                                      | Base OS support status                                                       | Non-kernel vulns | Critical / High | Fix available | Update path I found                             |
| ---------------------------------------------- | -------------------------------------------- | ---------------------------------------------------------------------------- | ---------------: | --------------: | ------------: | ----------------------------------------------- |
| **Linutop OS** (Jan 2025)                      | Raspbian 10                                  | **End of life** (Debian LTS ended June 2024)                                 |      318 (756\*) |        28 / 151 |           316 | Plain apt against the archived Debian 10 repo   |
| **XOGO** 2.105.21 (Oct 2022)                   | balenaOS host, player container on Debian 10 | **End of life** (Debian 10; Alpine 3.15/3.16 supervisor containers also EoL) |      413 (547\*) |          3 / 37 |           413 | balena fleet updates, if the fleet is managed   |
| **3DPrinterOS** (Mar 2024)                     | Arch Linux ARM snapshot, kernel 6.6.21       | Rolling release, 20 months stale                                             |             299† |         10 / 47 |           299 | Vendor service; no evidence of OS-level updates |
| **info-beamer OS** (Feb 2026)                  | Custom minimal Linux                         | Runtime on **Python 2.7.18** (EoL Jan 2020)                                  |              117 |         19 / 60 |           116 | Signed A/B image updates                        |
| **1Play** V17 (Jul 2025)                       | Raspbian 12.11                               | Debian 12 is in LTS since July 2026                                          |            3,877 |     257 / 1,371 |         2,617 | apt against Raspbian and Raspberry Pi repos     |
| **Yodeck** (Aug 2026, Pi 5)                    | Debian 12.5                                  | Debian 12 LTS; packages from the March 2024 point release                    |            3,266 |     208 / 1,201 |         2,643 | Vendor apt mirror                               |
| **TLXOS** 5.2.4                                | Debian 12.13                                 | Debian 12 LTS                                                                |            6,273 |     598 / 2,581 |         5,252 | Vendor updater service + Debian security repo   |
| **Homey Self-Hosted Server** (Dec 2025)        | Debian 13.2                                  | Supported                                                                    |            1,397 |        59 / 429 |         1,103 | Debian security repo, app in Docker             |
| **ScreenlyOS** 26.10.1 (Nov 2026)              | Ubuntu Core 24/26                            | Supported                                                                    |              673 |        37 / 148 |           349 | Transactional snap updates (automatic)          |
| **Android by emteria** 17.2.0 (Nov 2026, Pi 5) | Android 17                                   | Security patch level **2026-06-05**, five months behind its own build date   |             n/a‡ |             n/a |           n/a | Vendor OTA                                      |
| **Fugo One**                                   | n/a                                          | n/a                                                                          |              n/a |             n/a |           n/a | Image URL in Imager returns 404                 |
| **vetra player**                               | n/a                                          | n/a                                                                          |              n/a |             n/a |           n/a | Listed in Imager with no images                 |

\* The same packages, checked against Debian 11 advisory data; see "Scanners go quiet exactly when you need them" below.
† Only the application layer could be scanned. OSV has no Arch Linux (ALPM) ecosystem, so none of the 386 OS packages were checked.
‡ I read the Android build properties. I did not build a full Android SBOM.

Full disclosure before we go further: ScreenlyOS is made by Screenly, the company I co-founded. It is on the list, so it went through the exact same pipeline as everyone else. It does not come out clean either, and I left it in the table precisely so you can hold me to the same standard.

## What stands out

### End-of-life bases are still shipping

Two of the nine images I could fully analyse are built on Debian 10. Regular security support for Debian 10 ended in September 2022 and LTS in June 2024. Linutop's image, dated January 2025, is Raspbian 10.13 on a 5.10.103 kernel from early 2022, and its package database was last touched in November 2024. XOGO's player runs in a Debian 10 container on a balenaOS host with a 5.10.95 kernel, and its supervisor containers are built on Alpine 3.15 and 3.16, both also past end of life.

info-beamer is a more interesting case. Its base is a small custom build with signed A/B updates, which is genuinely the right architecture. But the device management stack runs on Python 2.7.18 with Flask 0.12, Werkzeug 0.11, Jinja2 2.8 and Pillow 4.0. Python 2 died in January 2020, and those libraries are from 2016-2017. Pillow alone accounts for 70 of the 117 advisories.

3DPrinterOS uses Arch Linux ARM, a rolling release. That is fine as long as you keep rolling. A March 2024 snapshot is just a 20-month-old Linux system with no versioned distribution behind it to backport fixes.

### "Supported" doesn't mean patched

The Debian 12 images look better on paper, because bookworm is still in LTS. In practice they carry the most vulnerabilities, because they ship a lot of software and the image is a frozen snapshot:

- **Yodeck's** Pi 5 image is dated August 2026 but built from Debian 12.5, which shipped in February 2024. Chromium is at 125, from June 2024. 2,643 of its 3,266 non-kernel advisories already have a fix in Debian.
- **1Play** ships two complete kernel trees (6.12.25 and 6.12.34) for every Pi generation, Chromium 138, and 2,617 fixable advisories.
- **TLXOS** is the most current Debian 12 image of the lot (12.13), but it bundles an entire thin-client stack: Citrix, VMware Horizon, FreeRDP, NX, SPICE, CUPS, strongSwan and more. Its Chromium build alone matches 4,206 advisories. Debian handles Chromium security differently from most packages, so take that one figure with a pinch of salt. The other 2,000-plus don't need one.

The same handful of packages dominates every Debian-based image: `webkit2gtk`, `imagemagick`, `ffmpeg`, `chromium`, `vim`, `binutils`, `pillow`. These are the media and browser components a signage player needs, and they are exactly the ones parsing untrusted content all day long.

### Scanners go quiet exactly when you need them

This one should bother anyone who trusts a scanner dashboard. When I scanned Linutop's Raspbian 10 packages against OSV's **Debian 10** data, I got **14** advisories. Against **Debian 11** data, the same packages matched **452**. XOGO went from 143 to 280.

The reason is mundane. Once a release is archived, the upstream security tracker stops recording new issues against it, so the vulnerability databases stop matching it. The older and less maintained a system gets, the cleaner its scan looks. A "0 critical" result on an end-of-life device is a false negative, not a clean bill of health. Checking the support status of the base OS has to be part of the job, not just running the scanner and reading the number.

### A default password and shared SSH host keys

A CVE count tells you what software a vendor pulled in. It says nothing about how they configured the thing. So I also looked at two items an SBOM never catches: the accounts baked into the image, and the SSH host keys.

The worst offender is **Linutop**. Its image ships with the `pi` account still set to the default Raspberry Pi password, `raspberry` -- the password Raspberry Pi OS itself stopped shipping in 2022 precisely because it was the single most exploited default on the platform. Anyone who can reach the device and knows the default, which is everyone, is already in. That is a hardcoded credential in the plainest possible sense.

The SSH story is separate and hits more images. Several vendors bake the **private SSH host keys** straight into the image, so every device flashed from it presents the _same_ key:

| Product         | SSH on by default   | Host keys               | First-boot regeneration    | Password login                                    |
| --------------- | ------------------- | ----------------------- | -------------------------- | ------------------------------------------------- |
| **Linutop**     | No (needs enabling) | **Baked in, shared**    | None                       | Default `pi` / `raspberry`                        |
| **1Play**       | Yes                 | **Baked in, shared**    | None                       | `pi` set (not default)                            |
| **TLXOS**       | Yes                 | **Baked in, shared**    | None                       | Disabled (key-only, root `prohibit-password`)     |
| **Yodeck**      | Yes                 | Generated on first boot | `regenerate_ssh_host_keys` | `root` / `pi` / `user` set; `PermitRootLogin yes` |
| **Homey**       | Yes                 | Generated on first boot | `regenerate_ssh_host_keys` | `homey` set                                       |
| **3DPrinterOS** | Yes                 | None in image           | n/a                        | `root` / `printeros` set                          |

A shared private host key is a real problem, not a cosmetic one. Every device in the field holds the identical key, so the key is only as secret as the image download, which is public. Pull the image and you own the private key for every deployed unit. That kills SSH's protection against man-in-the-middle interception: a box on the same network can impersonate the real player, and the admin connecting in gets no host-key warning, because the key matches. The fix is trivial -- delete the keys at build time and let `ssh-keygen` run once per device on first boot, which Yodeck and Homey already do -- which is what makes its absence everywhere else so annoying.

To be clear about what I did and didn't extract: `pi` / `raspberry` is the platform's own documented default, so naming it gives away nothing that isn't already public. I did not crack, and am not publishing, the unique password hashes on the other images, nor any private key material. The finding is simply that the keys are there and shared, which you can see without ever reading them. Credit where it's due: TLXOS disables password authentication entirely, though its shared host key still stands.

### Update mechanisms are all over the map

| Approach                                                 | Products                                        |
| -------------------------------------------------------- | ----------------------------------------------- |
| Transactional / image-based OTA (A/B or snaps)           | ScreenlyOS, info-beamer, XOGO (balena), emteria |
| Package manager pointed at live security repos           | TLXOS, Homey, 1Play                             |
| Package manager pointed at a vendor-controlled mirror    | Yodeck                                          |
| Package manager pointed at an archived, end-of-life repo | Linutop                                         |
| No OS-level update path visible in the image             | 3DPrinterOS                                     |

Image-based OTA is the right architecture for an appliance, but it only helps if the vendor actually rebuilds the image with current components. XOGO has balena's update machinery and ships a Debian 10 payload through it. info-beamer has signed A/B updates and ships Python 2.7 through them. And a package manager pointed at live Debian security repos only helps if something actually runs the upgrade. In none of the Debian-based images did I find an unattended-upgrades policy configured.

### The part nobody mentions: apt can't cross a Debian release

This is the detail that turns "end of life" from a slow-moving worry into a brick wall, and it is the most important thing to understand about every package-manager row above. `apt upgrade` keeps a device current _within_ one Debian release. It does not, in any realistic fleet sense, move a device _between_ releases.

A Debian major upgrade -- 11 to 12, or 12 to 13 -- is a full `dist-upgrade`: thousands of packages swapped, maintainer scripts run, config files merged, the C library and init system replaced underneath a running system. On my laptop, with a keyboard and a screen and the ability to answer a debconf prompt, it is a supported but delicate afternoon. On a headless signage player bolted behind a screen in a shop, pushed unattended to a whole fleet, with no console and no way to step in when a maintainer script stops to ask a question, it is not something any sane operator attempts. One interrupted upgrade -- a power cut, a dropped network, a single package conflict -- and you have a bricked device that needs a site visit anyway. Nobody ships a cross-release `dist-upgrade` as an OTA strategy, and that is the whole point.

So for every Debian-based product here, the base OS has a hard expiry date. Debian 12 gets security support into 2026 and LTS into mid-2028; Debian 10 is already past it. When that date lands, the apt channel these devices lean on simply stops getting fixes, and there is no in-place path to the next release. At that point the device isn't "a bit behind." It is permanently frozen on an unsupported OS, piling up unpatched CVEs for the rest of its physical life, and the only way forward is a reflash: someone physically re-imaging each unit, or the vendor pushing an entirely new image through a channel that, for the apt-based products, doesn't exist today.

This is where the transactional and image-based vendors have a real structural edge. A/B image OTA and snap systems (ScreenlyOS, info-beamer, emteria, balena) can ship a whole new base OS as just another signed image -- crossing a release boundary is the same atomic operation as any other update, with a rollback if it fails. The apt-based products (Yodeck, TLXOS, 1Play, Homey, Linutop) have no equivalent. For them, the base OS reaching end of life is a truck roll, not a download, and that cost is exactly why, in practice, it usually never happens.

## Why I actually care: this stuff lives on your network

Every one of these devices is a Linux computer with network access, physically sitting in a public or semi-public space, installed by facilities or marketing rather than IT, and left alone for years once it works. Many run a full browser engine rendering remote content. Several bundle VPN clients, remote-desktop clients and printing stacks. A compromised signage player is a quiet foothold inside the building's network that nobody is watching.

This isn't theoretical. Attackers already go after unmanaged IoT and edge devices to get initial access, precisely because those boxes are the least likely to be patched or monitored. Thousands of fixable vulnerabilities in a browser and media stack, on a device that fetches content from the internet and sits next to the corporate LAN, is the exact profile they look for.

## And then there's the CRA

These are commercial products sold in the EU, so the [Cyber Resilience Act](https://sbomify.com/compliance/eu-cra/) applies to them. The main obligations kick in on **11 December 2027**, and the reporting obligations already started on **11 September 2026**. Lined up against what I found:

- **No known exploitable vulnerabilities at release** (Annex I, Part I(2)(a)). Shipping a 2026 image built on a 2024 point release, or on an end-of-life base, is hard to square with this.
- **Security updates, automatic by default** (Annex I, Part I(2)(c)). An image with no working update path, or one pointed at an archived repo, doesn't meet it.
- **A defined support period, normally at least five years** (Article 13(8)). It has to cover the base OS, and you cannot promise five years of support on a distribution that is already end of life.
- **An SBOM covering at least the top-level dependencies** (Annex I, Part II(1)). Every vendor here would need one. Generating it is the easy bit. Acting on it is where they fall down.
- **Handling vulnerabilities without delay** (Annex I, Part II(2)). Two to five thousand already-fixed advisories in a shipping image is not what "without delay" looks like.
- **No default or unchangeable passwords, and protection from unauthorised access** (Annex I, Part I(2)(b)). A shipped `pi` / `raspberry` default and shared SSH host keys are the textbook failures here.

None of this needs heroics. Most of the problems above disappear if you rebuild the image on a supported base on a schedule, ship it through an update mechanism the device actually uses, and keep watching the SBOM for new vulnerabilities as they land. That last part -- continuously watching the SBOM rather than scanning once at build time -- is the problem I spend my days on at [sbomify](https://sbomify.com), so take that recommendation with the appropriate grain of salt. The first two are just engineering discipline.

## Takeaways

**If you buy these things**, ask the vendor for an SBOM and a stated support period, and ask how OS-level updates actually reach the device. Put signage players on an isolated network segment, and put them in your asset inventory.

**If you build them**, treat the base image as a dependency with an end-of-life date, and remember apt won't save you when that date arrives. There is no realistic cross-release OTA, so plan an image-based update path _before_ your base OS expires, not a fleet-wide reflash after. Track the base the way you track your application dependencies. Rebuild regularly, ship through a channel the device uses, and don't trust a clean scan of an end-of-life system. And handle the basics an SBOM can't see: no default passwords, and regenerate SSH host keys on first boot instead of baking one shared key into every unit.

**For everyone else**: a hobby image that's two years old is a hobby project. A commercial product that's two years behind is a liability -- and from December 2027, a compliance problem too. Turns out "pretty terrible" was generous.
