---
layout: post
title:  "Spectrum Sharing in LTE"
description: An introduction to LTE, LTE-Advanced and Spectrum Sharing
img:
date: 2018-08-18  +1835
---

### LTE - Long Term Evolution
+ Standard for high-speed wireless communication for mobile devices.
+ Based on GSM/EDGE and UMTS/HSPA technology.
+ Common standard for paired and unpaired spectrum.
+ LTE-FDD and LTE-TDD - 2 modes, common standards, same ecosystem
	- Standard is almost same for FDD and TDD.
	- Enables common FDD/TDD products.
+ LTE-FDD (Frequency Division Duplex)
	- Uses paired frequencies to upload and download data.
	- Can cover larger area - only applicable to coverage driven deployments.
	- Fixed uplink/downlink on different frequencies.
+ LTE-TDD (Time Division Duplex)
	- Uses a single frequency, alternating through time to upload and download data.
	- Can assign more downlink capacity - flexibility to assign more resources to meet asymmetric data usage.
	- Flexible uplink/downlink ratio.

### LTE Advanced
- Better, faster mobile broadband experience
+ Carrier Aggregation
	- Combines multiple LTE carriers for wider bandwidth, eg. fatter pipe
	- Enable mobile operators to maximize use of spectrum assets
	+ Aggregation across more carriers
	+ Aggregation across diverse spectrum types
		- Makes the best use of spectrum, eg. FDD/TDD, unlicensed/licensed spectrum.
+ Advanced MIMO (Multiple Input, Multiple Output)
	- Leverages more antennas to increase spectral efficiency.
+ Using higher order modulation: 256-QAM
	- Increases spectral efficiency for faster downlink throughput.
	- Increase bits per transmission (symbol)
+ Heterogenous Networks (HetNets)
	+ Small cells to efficiently increase capacity
	+ Reuse spectrum assets
		- Repeating Shannon's Law everywhere
	+ Provide localized capacity
		- For better signal quality, especially indoors
	+ Use unlicensed spectrum
		- Opportunistically while maintaining anchor in licensed spectrum
	+ LTE Solutions for enhancing HetNets
		+ Interference Management - capacity scales with small cells added
		+ Best use of all spectrum - unlicensed spectrum, shared licensed and higher bands
			+ 5GHz unlicensed spectrum
				- ideal for small cells due to low mandated transmit power
				- large amounts of spectrum available
		+ Self organizing networks - Plug and Play
		+ Dual connectivity
		+ Coordinated Multi-point

### Spectrum Sharing
2 or more systems operate in the same band - improving overall spectrum usage efficiency
+ Authorized Shared Access
+ **Licensed Shared Access**
	- dynamic use of a spectrum wherever and whenever it is unused by the incumbent spectrum user on a voluntary basis.
	- sharing could be done if incumbent uses only a subset of its frequency band, does not use the spectrum continuously and/or uses the spectrum only in a geographically limited area.

### Spectrum Sharing in unlicensed spectrum
- Spectrum is a scarce resource.
- Design spectrum sharing rules and protocols in a fair, efficient way compatible with incentives of individual systems.
- Resource allocation is efficient if it is not possible to improve performance of a given system without degrading performance of another system.

### Terminology and abbreviations
+ CRS: Cognitive Radio Sensing
	+ A cognitive radio is a radio that can be programmed and configured dynamically to use the best wireless channels in its vicinity to avoid user interference and congestion.
	+ This is a form of dynamic spectrum management.
+ MNO: Mobile Network Operator
