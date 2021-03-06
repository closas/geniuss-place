---
title: "Configurations"
permalink: /conf/
excerpt: "How to configure GNSS-SDR in a variety of setups."
related: true
last_modified_at: 2016-04-13T15:54:02-04:00
header:
  teaser: "http://exploreflask.com/en/latest/_images/configuration.png"
sidebar:
  nav: "start"
---

{% include toc %}

Obtaining position fixes from a file is nice and useful, but the real deal for a software-defined receiver is to play with live GNSS signals in real-time. This page describes examples of hardware setups, software configurations and general tips for obtaining position fixes (and a collection of side data, delivered in standard formats) with GNSS-SDR.


**Important note:** Either if you are capturing signal in a file for post-processing, or if you are configuring a receiver to work in real-time with live signal samples delivered by a radio frequency front-end, **you will need an active antenna** (that is, an antenna with an integrated low noise amplifier delivering a gain in the band of interest greater than $$ 20 $$ dB, with a noise figure below $$ 2 $$ dB), **and you will need to feed it**.  
{: .notice--warning}

Some radio frequency front-ends have jumpers, or some other configurable mode, for antenna feeding. If this feature is not available, you will need a bias-T in the antenna cable or some other kind of protection in order to inject power to the antenna without harming the front-end, plus an adequate power source (GNSS antennas use to be fed in a range from 2.5 VDC to 5.5 VDC, please check your model's datasheet).

## GPS L1 C/A receiver using a USRP

This in an example of an eight-channel GPS L1 C/A receiver, working at 4 MS/s (baseband, _i.e._ complex samples), and using a device from the [USRP family](https://www.ettus.com/product){:target="_blank"} as the "air-to-computer" interface.

### Required equipment

In order to get real-time position fixes, you will need:

  * **An active GPS antenna**. Any model will fit, just check that you can plug it to the USRP's SMA-male connector.
  * **A USRP**. All models will do the job. Depending on the specific USRP model you are using, the configuration may vary (see below).

  |----------
  |  **USRP Model**  |  **RF bandwidth (MHz @ 16 bits per sample)** | **RX gain (dB)** | **ADC Processing Bandwidth (MS/s)** | **Interface** | **Host Sample Rate (MS/s @ 16-bit I&Q)** |
  |:--|:-:|:-:|:-:|:-:|:-:|
  |--------------
  | USRP 1, B100 | Up to $$ 8 $$ MHz | See daughterboard | $$ 64 $$ MS/s | USB 2.0 | $$ 8 $$ MS/s |
  | B200, B210, B200mini, B205mini | From $$ 200 $$ kHz to $$ 56 $$ MHz |  Up to $$ 73 $$ dB | $$ 61.44 $$ MS/s | USB 3.0 |  $$ 61.44 $$ MS/s |
  | N200, N210 |  Up to $$ 25 $$ MHz | See daughterboard | $$ 100 $$ MS/s | Gigabit Ethernet | $$ 25 $$ MS/s |
  | X300, X310 | Up to $$ 120 $$ MHz | See daughterboard |  $$ 200 $$ MS/s | 10 Gigabit Ethernet | $$ 200 $$ MS/s |
  |--------------

  _Some USRP models and features. The ADC processing bandwidth is the sample rate provided by the ADCs on the USRP motherboard, and the host sample rate refers to the sample stream between the FPGA of a USRP device, and a host PC. Some USRP models also provide to option to stream 8-bit samples, effectively doubling the host-bandwidth in terms of samples/second. Source: [Ettus Research Knowledge Base](https://kb.ettus.com/About_USRP_Bandwidths_and_Sampling_Rates){:target="_blank"}._
  {: style="text-align: center;"}

  * The USRP family features a modular architecture with interchangeable daughterboard modules that serve as the RF front end. In case of using a USRP without an embedded transceiver, you will need **a daughterboard** allowing the reception of signals around 1.5 GHz. That is: DBSRX2, WBX, SBX, CBX and UBX daughterboards can work for you. You will not need a daughterboard if you are using USRP B200, B210 or E310, which ship an Analog Devices AD9361 RFIC as an integrated wideband transceiver.

  |----------
  |  **Daughterboard**  |  **Frequency coverage** | **Analog bandwidth** | **RX gain**
  |:--|:-:|:-:|:-:|
  |--------------
  | WBX-120 | 50 MHz - 2.2 GHz | 120 MHz | $$ 0 $$ to $$ 31.5 $$ dB |
  | SBX-120 | 400 MHz - 4.4 GHz | 120 MHz | $$ 0 $$ to $$ 31.5 $$ dB |
  | CBX-120 | 1.2 GHz - 6 GHz | 120 MHz | $$ 0 $$ to $$ 31.5 $$ dB |
  | UBX-160 | 10 MHz - 2.2 GHz | 160 MHz | $$ 0 $$ to $$ 31.5 $$ dB |
  | WBX | 50 MHz - 2.2 GHz | 40 MHz | $$ 0 $$ to $$ 31.5 $$ dB |
  | SBX | 400 MHz - 4.4 GHz | 40 MHz | $$ 0 $$ to $$ 31.5 $$ dB |
  | CBX | 1.2 GHz - 6 GHz | 40 MHz | $$ 0 $$ to $$ 31.5 $$ dB |
  | UBX-40 | 10 MHz - 6 GHz | 40 MHz | $$ 0 $$ to $$ 31.5 $$ dB |
  | DBSRX2 | 800 MHz - 2.3 GHz | Configurable: 8 MHz to 80 MHz | GC1 from $$ 0 $$ to $$ 73 $$ dB, and BBC from $$ 0 $$ to $$ 15 $$ dB |
  |--------------

  _Daugtherboards allowing GNSS signal reception. Note that WBX-120, CBX-120 and SBX-120 daughterboards were designed to work with USRP X300/X310 and any future products with sufficient ADC/DAC sample rates. These boards are not compatible with devices that incorporate lower rate ADC/DAC below 200 MS/s. Source: [Ettus Research Knowledge Base](https://kb.ettus.com/About_USRP_Bandwidths_and_Sampling_Rates){:target="_blank"}._
  {: style="text-align: center;"}


  * **A computer** connected to the USRP and with GNSS-SDR installed.

### Setting up the front-end

The first thing to do is to install a suitable daughterboard into the USRP. As a example, you can check Ettus Research's detailed [step-by-step guide to install a daughterboard into the USRP N200/N210](https://kb.ettus.com/USRP_N_Series_Quick_Start_(Daughterboard_Installation)){:target="_blank"}. In USRPs with two receiving slots, please check in which one you are inserting the daughterboard (they are usually labelled as "RX A" and "RX B"). This is something that you will need to specify in the configuration file (via the `subdevice` parameter, see below).

Then, you will need to feed your GNSS active antenna.

**Important:** Never apply more than -15 dBm of power into any RF input.
{: .notice--danger}

In case of using a DBSRX2 daughterboard, you will need to adjust the J101 jumper in order to feed the antenna.

![DBSRX2](http://yo3iiu.ro/blog/wp-content/uploads/2013/02/DBSRX2_scale.jpg){: .align-center}
_DBSRX2 daughterboard. The J101 jumper in the upper right corner allows the injection of current towards the antenna. Source: [Radio Adventures](http://yo3iiu.ro/blog/){:target="_blank"}._
{: style="text-align: center;"}

If this feature is not available (_e.g._, WBX daughterboards), you will need a bias-T between the USRP and the antenna, and to connect it to a power source delivering the voltage required by your antenna (usually, 3 V or 5 V).

![Bias-T](http://yo3iiu.ro/blog/wp-content/uploads/2013/02/bias_tee_scale.jpg){: .align-center}
_Bias-T allowing the injection of DC voltage to the antenna. Source: [Radio Adventures](http://yo3iiu.ro/blog/){:target="_blank"}._
{: style="text-align: center;"}


![Setup](http://yo3iiu.ro/blog/wp-content/uploads/2013/02/whole_hw_scaled.jpeg){: .align-center}
_USRP N210 with the bias-T and the GPS antenna. Source: [Radio Adventures](http://yo3iiu.ro/blog/){:target="_blank"}._
{: style="text-align: center;"}

Depending on the specific USRP model you are using, the connection to the host computer in charge of the execution of the software receiver can be through USB (2.0 or 3.0) or Ethernet (1 GigE or 10 GigE). Once connected, every USRP device has [several ways of identifying it](http://files.ettus.com/manual/page_identification.html){:target="_blank"} on the host system:

|----------
|  **Identifier**  |  **Key** | **Notes** | **Example** |
|:--|:--|:--|:--|
|--------------
| Serial #	| `serial` | 	globally unique identifier | 	12345678 |
| IP Address | `addr` |	unique identifier on a network	| 192.168.10.2 |
| Type | `type` | 	hardware series identifier | 	usrp1, usrp2, b200, x300, ... |
| Name | `name` | optional user-set identifier | lab1_usrp3 |
|--------------

 _Common device identifiers. Source: [Identifying USRP Devices](http://files.ettus.com/manual/page_identification.html){:target="_blank"}._
 {: style="text-align: center;"}

Devices attached to your system can be discovered using the `uhd_find_devices` program. This program scans your system for supported devices and prints out an enumerated list of discovered devices and their addresses. If you type `uhd_find_devices --help` in a terminal, you should see something similar to this:

```
$ uhd_find_devices --help
linux; GNU C++ version 4.9.2; Boost_105400; UHD_003.010.git-0-2d68f228

UHD Find Devices Allowed options:
  --help                help message
  --args arg            device address args
```

Then, you can search your USRP in a specific IP address:

```
$ uhd_find_devices --args addr=192.168.50.2
linux; GNU C++ version 4.9.2; Boost_105400; UHD_003.010.git-0-2d68f228

--------------------------------------------------
-- UHD Device 0
--------------------------------------------------
Device Address:
    type: x300
    addr: 192.168.50.2
    fpga: HGS
    name:
    serial: F5CA38
    product: X300
```

or by typing:

```
$ uhd_find_devices --args type=usrp1
```

This is a good way to check if the USRP is correctly connected to your computer. After this check, we can proceed to configure the software receiver. The [USRP Hardware Driver and USRP Manual](http://files.ettus.com/manual/page_devices.html){:target="_blank"} provides more information about the configuration and usage of those devices.

### Setting up the software receiver

Copy the configuration below into you favorite plain text editor and save it with a name such as `my_GPS_receiver.conf` in your favorite working directory.

```ini
;######### GLOBAL OPTIONS ##################
GNSS-SDR.internal_fs_hz=4000000

;######### SIGNAL_SOURCE CONFIG ############
SignalSource.implementation=UHD_Signal_Source
SignalSource.device_address=192.168.50.2 ; <- PUT THE IP ADDRESS OF YOUR USRP HERE
                                         ;    OR LEAVE IT EMPTY FOR USB
SignalSource.item_type=cshort
SignalSource.sampling_frequency=4000000
SignalSource.freq=1575420000
SignalSource.gain=40
SignalSource.subdevice=A:0  ; <- Can be A:0 or B:0
SignalSource.samples=0

;######### SIGNAL_CONDITIONER CONFIG ############
SignalConditioner.implementation=Signal_Conditioner

;######### DATA_TYPE_ADAPTER CONFIG ############
DataTypeAdapter.implementation=Pass_Through
DataTypeAdapter.item_type=cshort

;######### INPUT_FILTER CONFIG ############
InputFilter.implementation=Fir_Filter
InputFilter.input_item_type=cshort
InputFilter.output_item_type=gr_complex
InputFilter.taps_item_type=float
InputFilter.number_of_taps=11
InputFilter.number_of_bands=2

InputFilter.band1_begin=0.0
InputFilter.band1_end=0.48
InputFilter.band2_begin=0.52
InputFilter.band2_end=1.0

InputFilter.ampl1_begin=1.0
InputFilter.ampl1_end=1.0
InputFilter.ampl2_begin=0.0
InputFilter.ampl2_end=0.0

InputFilter.band1_error=1.0
InputFilter.band2_error=1.0

InputFilter.filter_type=bandpass
InputFilter.grid_density=16
InputFilter.sampling_frequency=4000000
InputFilter.IF=0

;######### RESAMPLER CONFIG ############
Resampler.implementation=Pass_Through

;######### CHANNELS GLOBAL CONFIG ############
Channels_1C.count=8
Channels.in_acquisition=1
Channel.signal=1C

;######### ACQUISITION GLOBAL CONFIG ############
Acquisition_1C.implementation=GPS_L1_CA_PCPS_Acquisition
Acquisition_1C.threshold=0.01
Acquisition_1C.doppler_max=8000
Acquisition_1C.doppler_step=500

;######### TRACKING GLOBAL CONFIG ############
Tracking_1C.implementation=GPS_L1_CA_DLL_PLL_Tracking
Tracking_1C.pll_bw_hz=30.0
Tracking_1C.dll_bw_hz=4.0
Tracking_1C.early_late_space_chips=0.5

;######### TELEMETRY DECODER GPS CONFIG ############
TelemetryDecoder_1C.implementation=GPS_L1_CA_Telemetry_Decoder

;######### OBSERVABLES CONFIG ############
Observables.implementation=GPS_L1_CA_Observables

;######### PVT CONFIG ############
PVT.implementation=GPS_L1_CA_PVT
PVT.averaging_depth=10
PVT.flag_averaging=true
PVT.output_rate_ms=100
PVT.display_rate_ms=500
PVT.flag_rtcm_server=true
```

You will need to adjust the values for at least two parameters:

  * Check that `SignalSource.device_address` points to the actual IP address of your USRP, if you are connected through Ethernet, or leave it empty for USB.
  * Check that `SignalSource.subdevice` is set to the receiving slot in which you actually inserted your daughterboard with the antenna. In USRPs with only one receiving slot, leave it as `A:0`. Please check more details on [how to specify the subdevice](http://files.ettus.com/manual/page_configuration.html#config_subdev){:target="_blank"}.

The [Signal Processing Blocks documentation]({{ "/docs/sp-blocks/" | absolute_url }}){:target="_blank"} provides definitions and more details about the configuration parameters.

### Run it!

Once the hardware and the software configurations are ready, go to your favorite working directory where the file `my_GPS_receiver.conf` was stored and invoke the software receiver with this particular configuration:

```bash
$ gnss-sdr --config_file=./my_GPS_receiver.conf
```

You should see something similar to:

```
$ gnss-sdr --config_file=./my_GPS_receiver.conf
linux; GNU C++ version 4.9.2; Boost_105400; UHD_003.010.git-0-2d68f228

Initializing GNSS-SDR v0.0.9 ... Please wait.
Logging will be done at "/tmp"
Use gnss-sdr --log_dir=/path/to/log to change that.
-- X300 initialization sequence...
-- Determining maximum frame size... 8000 bytes.
-- Setup basic communication...
-- Loading values from EEPROM...
-- Setup RF frontend clocking...
-- Radio 1x clock:200
-- Initialize Radio0 control...
-- Performing register loopback test... pass
-- Initialize Radio1 control...
-- Performing register loopback test... pass
Sampling Rate for the USRP device: 4000000.000000 [sps]...
UHD RF CHANNEL #0 SETTINGS
Actual USRP center freq.: 1575420000.010133 [Hz]...
PLL Frequency tune error 0.010133 [Hz]...
Actual daughterboard gain set to: 37.500000 dB...
Setting RF bandpass filter bandwidth to: 2000000.000000 [Hz]...
Check for front-end LO: locked ... is Locked
Using Volk machine: avx2_64_mmx
Starting a TCP Server on port 2101
The TCP Server is up and running. Accepting connections ...
...
```

Of course, file `my_GPS_receiver.conf` can be wherever (`--config-file` accepts both relative and absolute paths), and the data displayed at the terminal output might vary according to your setup.

Then, after some seconds detecting GPS signals and decoding some frames of their navigation messages (at least, subframes 1, 2 and 3 from four satellites)...

```

...
Current input signal time = 49 [s]
Current input signal time = 50 [s]
Current input signal time = 51 [s]
Current input signal time = 52 [s]
NAV Message: received subframe 1 from satellite GPS PRN 27 (Block IIF)
NAV Message: received subframe 1 from satellite GPS PRN 10 (Block IIF)
NAV Message: received subframe 1 from satellite GPS PRN 08 (Block IIF)
NAV Message: received subframe 1 from satellite GPS PRN 16 (Block IIR)
NAV Message: received subframe 1 from satellite GPS PRN 18 (Block IIR)
Current input signal time = 53 [s]
Current input signal time = 54 [s]
Current input signal time = 55 [s]
Current input signal time = 56 [s]
Current input signal time = 57 [s]
Current input signal time = 58 [s]
NAV Message: received subframe 2 from satellite GPS PRN 27 (Block IIF)
NAV Message: received subframe 2 from satellite GPS PRN 10 (Block IIF)
NAV Message: received subframe 2 from satellite GPS PRN 08 (Block IIF)
NAV Message: received subframe 2 from satellite GPS PRN 16 (Block IIR)
NAV Message: received subframe 2 from satellite GPS PRN 18 (Block IIR)
Current input signal time = 59 [s]
Current input signal time = 60 [s]
Current input signal time = 61 [s]
Current input signal time = 62 [s]
Current input signal time = 63 [s]
Current input signal time = 64 [s]
NAV Message: received subframe 3 from satellite GPS PRN 27 (Block IIF)
NAV Message: received subframe 3 from satellite GPS PRN 10 (Block IIF)
NAV Message: received subframe 3 from satellite GPS PRN 08 (Block IIF)
NAV Message: received subframe 3 from satellite GPS PRN 16 (Block IIR)
NAV Message: received subframe 3 from satellite GPS PRN 18 (Block IIR)
Current input signal time = 65 [s]
Position at 2016-Aug-11 14:23:19 UTC is Lat = 41.2751 [deg], Long = 1.98765 [deg], Height= 68.9893 [m]
Position at 2016-Aug-11 14:23:19 UTC is Lat = 41.2751 [deg], Long = 1.98765 [deg], Height= 72.1068 [m]
Current input signal time = 66 [s]
Position at 2016-Aug-11 14:23:20 UTC is Lat = 41.2751 [deg], Long = 1.9877 [deg], Height= 67.0216 [m]
Position at 2016-Aug-11 14:23:20 UTC is Lat = 41.2751 [deg], Long = 1.9877 [deg], Height= 84.7445 [m]
Current input signal time = 67 [s]
Position at 2016-Aug-11 14:23:21 UTC is Lat = 41.2751 [deg], Long = 1.98771 [deg], Height= 70.0031 [m]
Position at 2016-Aug-11 14:23:21 UTC is Lat = 41.2751 [deg], Long = 1.98767 [deg], Height= 63.1242 [m]
Current input signal time = 68 [s]
...

```

  If you see something similar to this... Yay! You are getting real-time position fixes with your open source software-defined GPS receiver!
  {: .notice--success}

  **Important:** In order to get well-formatted GeoJSON, KML and RINEX files, always terminate ```gnss-sdr``` execution by pressing key '`q`' and then key '`ENTER`'. Those files will be automatically deleted if no position fix have been obtained during the execution of the software receiver.
  {: .notice--warning}

Always stop the execution of GNSS-SDR by pressing key '`q`'  and then key '`ENTER`' (_not_ at the same time, first '`q`' and then '`ENTER`'):


```

...
Position at 2016-Aug-11 14:23:31 UTC is Lat = 41.2751 [deg], Long = 1.98762 [deg], Height= 58.1987 [m]
q
Quit keystroke order received, stopping GNSS-SDR !!
Stopping GNSS-SDR, please wait!
Total GNSS-SDR run time 78.3891 [seconds]
GNSS-SDR program ended.
Stopping TCP Server on port 2101
$
```

Now you can examine the files created in your working folder.

### If something goes wrong

* Check out that the GNSS antenna is actually fed and that it is placed in a location with a good sky visibility.

* Check out Ettus Research's [getting starting guide](https://kb.ettus.com/Getting_Started_Guides){:target="_blank"} for your particular hardware configuration.

* Watch out for overflows! Maybe your host computer is not able to sustain the required computational load for this particular implementation.

{% capture overflow-text %}
 When receiving, the USRP device produces samples at a constant rate. Overflows occurs when the host computer does not consume data fast enough. When UHD software detects the overflow, it prints an "```O```" or "```D```" to the standard terminal output, and pushes an inline message packet into the receive stream.

  * **Network-based devices (_e.g._, USRP N2xx, X3xx)**: The host does not back-pressure the receive stream. When the kernel's socket buffer becomes full, it will drop subsequent packets. UHD software detects the overflow as a discontinuity in the packet's sequence numbers, and pushes an inline message packet into the receive stream. In this case the character ```D``` is printed to the standard terminal output as an indication.
  * **Other devices (_e.g._, USRP 1, B2xx)**: The host back-pressures the receive stream. Therefore, overflows always occur in the device itself. When the device's internal buffers become full, streaming is shut off, and an inline message packet is sent to the host. In this case the character "```O```" is printed to the standard terminal output as an indication. If the device was in continuous streaming mode, the UHD software will automatically restart streaming when the buffer has space again.
{% endcapture %}

<div class="notice--danger">
  <h4>Overflow warnings:</h4>
  {{ overflow-text | markdownify }}
</div>

 * Play with configuration parameters!


<link rel="prerender" href="{{ "/docs/overview/" | absolute_url }}">
<link rel="prerender" href="{{ "/docs/" | absolute_url }}">
