+++
date = '2025-06-08T19:41:25+02:00'
draft = true
title = 'Differential Amplifier for Oscilloscope'
+++


The probes in a typical oscilloscope are referred to a common ground. This is inconvenient in some situations, and makes measurement right out impossible in others. Differential voltage probes with adequate specifications are available for high voltages at a reasonable price. The situation does not seem to be the same for those of us who would like to measure low differential voltages, and especially low differential voltages in the presence of large common mode voltages. In the "cheap" segment thease types of probes easily costs > 50 kNOK (~5000 USD). 

Even in the situations where we are interested in ground referenced voltages it is a challenge to measure millivolts or lower on a typical oscilloscope.

There is often put a lot of emphasis on the bandwidth, and even the cheap high voltage probes are advertised with bandwidth of 100 MHz, or more. For a lot of applications this is not neccesary, and there are instead other parameters which are of higher importance.

Tektronix sells the ADA400A Differential Preamplifier at around 55 kNOK. It is advertised with the following specifications:

- 10 μV/div sensitivity
- Typically 100 dB CMRR DC to 10 kHz
- > 1 MHz bandwidth
- Noise typically ≤30 μVRMS at 100x gain

The maximum common mode voltage depends on the gain setting and is specified as +/- 10 V at the highest gain (when you want to measure differential voltages in the microvolt range). More information here: https://xdevs.com/fix/ada400a/

For comparison the relatively popular and cheap Micsig DP10013 100MHz 1300V differensial probe (which we have in our lab) has the following specifications:

- Noise referred to the input: ±40mVrms(50X), ±230 mVrms(500X）
- CMRR: ＞80dB(DC), ＞60dB(100KHz), ＞50dB(1MHz）
- Accuracy: 2%

As can be seen the Textronix CMMR is superior for the low frequencies where it is specified. Furthermore millivolt measurements will not be fun with the noise of the DP10013. Bandwidth is not everything.

Beware of the silly convention of using X to specify attenuation, 500X for the Micsig means divide by 500.

One of the reasons for the high price of differential amplifiers (probes) is probably relatively low demand, and also because they might require some manual tuning in a lab before they meet specifications. We decided we would like to attempt to design our own amplifier.

### Design specifications

ADA400A is a old design, and although it is our inspiration we would be happy to outperform it if modern components can compensate for the (likely) lower skill level we have compared to the Textronix engineers.

We prefer surface mounted components because of size, simpler PCB layout, and since THT components with good specifications are increasingly getting harder to find.

### Design and simulation

### Prototype and performance evaluation

