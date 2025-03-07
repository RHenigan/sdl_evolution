# SDL Evolution
**Do not submit a proposal without reading this roadmap document.** Ideas should be discussed on the [SDL Slack][sdl_slack] [#sdl_evolution][sdl_evolution_channel] open channel before a proposal is created here.

For more information about SDL, go to [www.smartdevicelink.com](https://www.smartdevicelink.com).

This process and document is based on Apple's excellent [Swift Evolution](https://github.com/apple/swift-evolution) process.

This document tracks the ongoing evolution of SDL. Proposals should be made for any public API surface change, that is, any change that would necessitate a minor or major version change in any SDL mobile library. If a protocol or RPC change does not change any API surface, they must still be brought through the proposal process. If changes are made to modify any remote control feature, only a single module should be addressed within a proposal. In sum, the following changes must be proposed:

* Changes necessitating minor or major version updates to SDL Core, the mobile libraries, Manticore, SDL Policy Server, and SHAID according to [semantic versioning](http://www.semver.org), except those that are caused by a misspelling.
* **Any** change to the SDL Protocol.
* **Any** change to the SDL RPC spec.

Important documents:

* Goals for upcoming SDL releases – this document.
* The [SDL evolution proposal review schedule](https://smartdevicelink.github.io/sdl_evolution/) to see upcoming SDL evolution proposal reviews.
* The [SDL evolution review process](process.md) to see how to propose and review proposals.
* The [proposals versus issues document](proposals_versus_issues.md) to understand the differences between an SDL Evolution Proposal and an SDL Issue.

## Platforms
When entering a proposal, only one platform will be listed as impacted on the [proposal status page][proposal-status].  It's inherent that some platforms impact others, as described below:

#### Protocol
The SmartDeviceLink protocol specification describes the method for establishing communication between an application and head unit and registering the application for continued communication with the head unit. The protocol is used as the base formation of packets sent from one module to another.

All new SDL implementations should implement the newest version of the protocol.

**Repository**: [https://github.com/smartdevicelink/protocol_spec](https://github.com/smartdevicelink/protocol_spec)

**Changes to Protocol also impact the following platforms**:

- Core
- iOS
- Java Suite
- JavaScript Suite

---

#### RPC Spec
Spec for RPC messages sent between mobile devices and hardware modules.

**Repository**: [https://github.com/smartdevicelink/rpc_spec](https://github.com/smartdevicelink/rpc_spec)

**Changes to RPC also impact the following platforms:**

- Core
- iOS
- Java Suite
- JavaScript Suite
- HMI (in most cases)

---

#### Core
The Core component is the software which Vehicle Manufacturers (OEMs) implement in their vehicle head units. Integrating this component into their head unit and HMI based on a set of guidelines and templates enables access to various applications.

**Repository**: [https://github.com/smartdevicelink/sdl_core](https://github.com/smartdevicelink/sdl_core)

**Changes to Core also impact the following platforms:**

- Manticore

---

#### Java Suite
The Android, Java SE, and Java EE libraries are implemented by app developers into their applications to enable command and control via the connected head unit. 

**Repository**: [https://github.com/smartdevicelink/sdl_java_suite](https://github.com/smartdevicelink/sdl_java_suite)

---

#### iOS SDK
The iOS library is implemented by iOS app developers into their applications to enable command and control via the connected head unit.

**Repository**: [https://github.com/smartdevicelink/sdl_ios](https://github.com/smartdevicelink/sdl_ios)

---

#### JavaScript Suite
The JavaScript Suite library is implemented by app developers into their JavaScript applications to enable command and control via the connected head unit.

**Repository**: [https://github.com/smartdevicelink/sdl_javascript_suite](https://github.com/smartdevicelink/sdl_javascript_suite)

---

#### Manticore 
Manticore is used to dynamically provision SDL Core and SDL HMI instances in the cloud.

**Repository**: [https://github.com/smartdevicelink/manticore](https://github.com/smartdevicelink/manticore)

**SDL hosted instance**: [https://smartdevicelink.com/resources/manticore/](https://smartdevicelink.com/resources/manticore/)

---

#### SDL Policy Server 
The SDL Policy Server handles authentication, data collection, and basic configurations for SDL connected vehicles.

**Repository**: [https://github.com/smartdevicelink/sdl_server](https://github.com/smartdevicelink/sdl_server)

---

#### SHAID 
SHAID is a centralized service designed to keep information about SDL-supported applications synchronized across the SmartDeviceLink (SDL) ecosystem.

**Documentation**: [SHAID Documentation](https://smartdevicelink.com/en/guides/shaid/v2/)

## Recent and Upcoming Releases
You can see a detailed list of accepted and implemented proposals for upcoming versions on the [proposal status page][proposal-status].  Each proposal is listed as only impacting one platform, as it's inherent that some platforms impact others, as described above in the `Platforms` Section.

## Recent Releases

#### Manticore: 2.8.0
**Release Date**: May 17, 2021

**Contents**: Release information can be found [here](https://github.com/smartdevicelink/manticore/releases/tag/2.8.0). 

---

#### SDL Protocol Spec: 5.4.1
**Release Date**: October 27, 2021

**Contents**: Release information can be found [here](https://github.com/smartdevicelink/protocol_spec/releases/tag/5.4.1).

---

#### SDL RPC Spec: 8.0.0
**Release Date**: October 27, 2021

**Contents**: Release information can be found [here](https://github.com/smartdevicelink/rpc_spec/releases/tag/8.0.0).

---

#### SDL Core: 8.0.0

**Release Date**: October 27, 2021

**Contents**: Release information can be found [here](https://github.com/smartdevicelink/sdl_core/releases/tag/8.0.0).

---

#### SDL iOS: 7.3.0
**Release Date**: October 27, 2021

**Contents**: Release information can be found [here](https://github.com/smartdevicelink/sdl_ios/releases/tag/7.3.0)

---

#### SDL Java Suite: 5.3.0
**Release Date**: October 27, 2021

**Contents**: Release information can be found [here](https://github.com/smartdevicelink/sdl_java_suite/releases/tag/5.3.0)

---

#### SDL JavaScript Suite: 1.5.0
**Release Date**: October 27, 2021

**Contents**: Release information can be found [here](https://github.com/smartdevicelink/sdl_javascript_suite/releases/tag/1.5.0).

---

#### SDL Policy Server: 2.12.0
**Release Date**: October 27, 2021

**Contents**: Release information can be found [here](https://github.com/smartdevicelink/sdl_server/releases/tag/2.12.0).

---

## Upcoming Releases

#### Manticore: 2.9.0
**Target Release**: November 2021

**Contents**: Contents of this release can be tracked on the [here](https://github.com/smartdevicelink/manticore/projects/4). 

---

#### SDL iOS: 7.4.0
**Target Release Date**: April 13, 2022

**Contents**: Contents of this release can be tracked [here](https://github.com/smartdevicelink/sdl_ios/projects/27).

---

#### SDL Java Suite: 5.4.0
**Target Release Date**: April 13, 2022

**Contents**: Contents of this release can be tracked [here](https://github.com/smartdevicelink/sdl_java_suite/projects/21).

---

#### SDL JavaScript Suite: 1.6.0
**Target Release Date**: April 13, 2022

**Contents**: Contents of this release can be tracked [here](https://github.com/smartdevicelink/sdl_javascript_suite/projects/7).

---

#### SDL Protocol Spec: 5.5
**Target Release Date**: April 13, 2022

**Contents**: Contents of this release can be tracked [here](https://github.com/smartdevicelink/protocol_spec/projects/7).

---

#### SDL RPC Spec: 8.1.0
**Target Release Date**: April 13, 2022

**Contents**: Contents of this release can be tracked [here](https://github.com/smartdevicelink/rpc_spec/projects/8).

---
#### SDL Core: 8.1.0
**Target Release Date**: April 13, 2022

**Contents**: Contents of this release can be tracked [here](https://github.com/smartdevicelink/sdl_core/projects/21).

---
#### SDL Policy Server: 2.13.0
**Target Release Date**: April 13, 2022

**Contents**: Contents of this release can be tracked [here](https://github.com/smartdevicelink/sdl_server/projects/7).

---

[sdl_slack]: http://slack.smartdevicelink.com "SDL Slack"
[sdl_evolution_channel]: https://smartdevicelink.slack.com/messages/sdl_evolution/ "sdl_evolution slack channel"
[proposal-status]: https://smartdevicelink.github.io/sdl_evolution/
