---
Description: Developing a WaveRT Miniport Driver
MS-HAID: 'audio.developing\_a\_wavert\_miniport\_driver'
MSHAttr: 'PreferredLib:/library/windows/hardware'
title: Developing a WaveRT Miniport Driver
---

# Developing a WaveRT Miniport Driver


This topic presents the software and hardware-related points you must consider when you decide to develop your own WaveRT miniport driver.

Microsoft has developed a set of hardware design guidelines for a [Universal Audio Architecture (UAA),](http://go.microsoft.com/fwlink/p/?linkid=30850) and the guidelines incorporate the features we recommend for a WaveRT audio device. The UAA guidelines are closely based on the High Definition (HD) Audio specification developed by Intel.

Windows Vista and later Windows operating systems provide an HD audio driver for UAA-compliant audio devices. So if your audio device is UAA-compliant, you do not have to develop your own WaveRT miniport driver. But for audio devices that have some proprietary, non-UAA hardware features, you must develop your own WaveRT miniport driver to support the proprietary features.

To help you to develop your own WaveRT miniport driver, we recommend that you first review the sample adapter driver, and then review the WaveRT-friendly UAA features.

### <span id="the_sample_adapter_driver"></span><span id="THE_SAMPLE_ADAPTER_DRIVER"></span>The sample adapter driver

For information on the sample driver, see [Sample Audio Drivers](sample-audio-drivers.md).

### <span id="the_wavert_friendly_features"></span><span id="THE_WAVERT_FRIENDLY_FEATURES"></span>The WaveRT-friendly features

After you review the sample adapter driver and start to design your WaveRT miniport driver, you must verify that it supports the following software and hardware features. As a result, the miniport driver that you build then becomes compatible with the system-supplied WaveRT port driver and with the mode of operation of the Windows Vista [audio engine](exploring-the-windows-vista-audio-engine.md).

-   **Low hardware latency.** A WaveRT miniport driver must provide a fully functioning implementation of the [**IMiniportWaveRTStream::GetHWLatency**](audio.iminiportwavertstream_gethwlatency) method. This method is necessary to support the [**KSPROPERTY\_RTAUDIO\_HWLATENCY**](audio.ksproperty_rtaudio_hwlatency) property.

-   **FIFO interrupts.** A WaveRT miniport driver must automatically generate interrupts when FIFO overruns and underruns occur. This feature allows the detection of glitches in the audio stream when you run tests on the audio device and driver software. Without hardware support (in other words, FIFO interrupts), no convenient and reliable method exists for obtaining glitch information.

-   **Scatter-Gather DMA and Buffer Looping.** When your miniport driver supports a DMA controller that has scatter-gather capabilities, it allows data to be moved into and out of the cyclic buffer without the need for intervention from your miniport driver.

    When your miniport driver supports a DMA controller that can perform buffer loops, the DMA controller can automatically wrap around to the start of the buffer after it reaches the end of the buffer with a read or write operation. It can perform the wrap around without intervention from your miniport driver.

    Note that the WaveRT port driver supports existing hardware designs that lack the ability to perform scatter-gather transfers or automatic buffer loops.

    If an audio device lacks scatter-gather capability, the WaveRT miniport driver must first allocate cyclic buffers that consist of pages that are physically contiguous in memory. The miniport driver then uses helper functions in the WaveRT port driver to perform the data transfers and automatic buffer looping. The drawback is that as the nonpaged memory pool of a system becomes increasingly fragmented, a request to allocate a large block of contiguous physical memory is more likely to fail. A device with scatter-gather capability is not affected by memory fragmentation.

    If an audio device cannot automatically perform buffer loops when the DMA channel reaches the end of the cyclic buffer, the WaveRT miniport driver must intervene and configure the channel to begin the transfer of data at the beginning of the buffer.

-   **Position Registers.** For new designs, hardware implementers should include a position register for each DMA channel. A position register indicates the current buffer position as a byte offset from the beginning of the cyclic buffer. The position register reading is zero at the beginning of the buffer. When the position register reaches the end of the cyclic buffer, it automatically wraps around to the beginning of the buffer (resets to zero) and continues to increment as the buffer position advances.

    Position registers can be mapped to virtual memory so that clients can read the registers directly.

    Ideally, position registers should indicate the buffer position of the samples that are currently moving through the digital-to-analog and analog-to-digital converters (DACs and ADCs) of the audio device.

    However, this information might not be directly available from an audio chipset that divides the digital and analog functions into separate bus-controller and encoder/decoder (codec) chips. Typically, the position registers are located in the bus-controller chip, and each register indicates the position of the audio data that the controller is writing to or reading from the codecs.

    After obtaining a reading from this type of position register, the client can estimate the current position of the samples that are moving through the DACs or ADCs by adding or subtracting the delay through the codec. The client obtains the codec delay from the **KSPROPERTY\_RTAUDIO\_HWLATENCY** property request. For this reason, a WaveRT miniport driver must accurately report the codec delay when the port driver calls the **IMiniportWaveRTStream::GetHardwareLatency** method in response to this type of property request.

    Note that the WaveRT port driver supports existing hardware designs that lack position registers. For a device with this limitation, the WaveRT miniport driver must fail calls to the [**IMiniportWaveRTStream::GetPositionRegister**](audio.iminiportwavertstream_getpositionregister) method by returning the **STATUS\_NOT\_SUPPORTED** error code, which forces the port driver to fail [**KSPROPERTY\_RTAUDIO\_POSITIONREGISTER**](audio.ksproperty_rtaudio_positionregister) property requests. In this case, clients must obtain the current position through the [**KSPROPERTY\_AUDIO\_POSITION**](audio.ksproperty_audio_position) property, which incurs the overhead of a transition between user mode and kernel mode for each position reading.

-   **Clock Register.** A clock register is an optional but useful hardware feature for a WaveRT-compatible audio device. Audio application programs can use clock registers to synchronize audio streams in two or more independent audio devices that have separate and unsynchronized hardware clocks. Without clock registers, an application is unable to detect and compensate for the drift between the hardware clocks.

    The sample clock that the audio hardware uses to clock audio data through the digital-to-analog or analog-to-digital converters should be derived from the internal clock that increments the clock register. A clock register that increments at a rate that is asynchronous with respect to the sample clock is of no use for synchronization and should not be exposed.

    Similar to the position registers, the clock register can be mapped to virtual memory so that clients can read the register directly.

-   **Audio Processing Objects.** A well-designed WaveRT miniport driver must never touch the audio data in the cyclic buffer of an audio device. The hardware should be designed so that audio data flows directly between the client and audio hardware with no intervention by the audio driver software. However, Windows Vista supports two types of audio processing objects (APOs) that perform software processing of audio data without violating this rule:

    -   Local effects (LFX) APOs

        LFX APOs perform generic audio processing functions (for example, equalization) that are not specific to a particular audio device. An LFX APO processes an audio stream from an application before the stream is added to the global mix.

    -   Global effects (GFX) APOs

        GFX APOs perform hardware-specific processing of an audio stream. A GFX APO is tied to a particular audio device by the INF file that installs the device. The effect of a GFX APO is global because it affects the global mix that plays through the audio device.

        **Note**   Windows Vista and later operating systems provide a set of APOs as part of the operating system. You can, however develop your own APOs to replace the system-supplied ones. For more information, see [System Effects Audio Processing Objects](system-effects-audio-processing-objects.md).

         

    Global mixing is performed by the audio engine, which is the user-mode system component that is responsible for mixing the audio streams from all audio applications. Typically, the audio engine is the client that directly exchanges data with the WaveRT audio device through the cyclic buffer.

    When the user enables an LFX APO, the audio system inserts the APO into one of the input streams to the audio engine. When the user enables a GFX APO, the system inserts that APO into the output stream from the audio engine. For more information about LFX and GFX APOs and the audio engine, see the [Exploring the Windows Vista Audio Engine](exploring-the-windows-vista-audio-engine.md) topic.

    APOs are available for use only with shared-mode audio streams. For exclusive-mode streams, applications exchange data directly with WaveRT hardware devices through cyclic buffers, and no other components can touch the data in the buffers.

 

 

[Send comments about this topic to Microsoft](mailto:wsddocfb@microsoft.com?subject=Documentation%20feedback%20[audio\audio]:%20Developing%20a%20WaveRT%20Miniport%20Driver%20%20RELEASE:%20%287/14/2016%29&body=%0A%0APRIVACY%20STATEMENT%0A%0AWe%20use%20your%20feedback%20to%20improve%20the%20documentation.%20We%20don't%20use%20your%20email%20address%20for%20any%20other%20purpose,%20and%20we'll%20remove%20your%20email%20address%20from%20our%20system%20after%20the%20issue%20that%20you're%20reporting%20is%20fixed.%20While%20we're%20working%20to%20fix%20this%20issue,%20we%20might%20send%20you%20an%20email%20message%20to%20ask%20for%20more%20info.%20Later,%20we%20might%20also%20send%20you%20an%20email%20message%20to%20let%20you%20know%20that%20we've%20addressed%20your%20feedback.%0A%0AFor%20more%20info%20about%20Microsoft's%20privacy%20policy,%20see%20http://privacy.microsoft.com/en-us/default.aspx. "Send comments about this topic to Microsoft")


