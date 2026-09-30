# A16_RISCV/external/v4l2_codec2: android16-riscv

Changes made for the Android 16 (AOSP, riscv64) bring-up of the BananaPi BPI-F3 (SpacemiT K1) and the BananaPi BPI-SM10 (SpacemiT K3), on branch `android16-riscv`.

Hardware video decode/encode on the SpacemiT Linlon-V5 (amvx) VPU.

## Changes

- **V4L2Decoder: prefer NV12 capture formats**: Output buffers are YCBCR_420_888, laid out as NV12 by minigbm. The SpacemiT K1 amvx driver lists YM12 first, so the decoder wrote 3-plane I420 into NV12 buffers and QBUF failed with EFAULT at 1080p.
- **V4L2Decoder: keep the provisional output format at least 128x128**: Drivers reporting tiny minimum sizes (amvx: 2x2) made the initial EOS buffer allocation fail in gralloc.
- **V4L2Decoder: free CAPTURE buffers before S_FMT on resolution change**: Follow the stateful decoder spec order (STREAMOFF, REQBUFS(0), S_FMT); amvx refuses S_FMT while buffers are allocated, failing every DRC.
- **V4L2Decoder: tolerate the initial EOS buffer on DQBUF**: amvx returns the frame-less initial EOS buffer flagged LAST at the first resolution change; the release-build no-op assert let the code erase(end()) and crash the codec service.
- **VideoFramePool: support the IGBA block pool**: Android 15+ AIDL Codec2 surface output uses IGraphicBufferAllocator blocks: read their dma-buf id like BUFFERQUEUE ones and retry when fetch returns C2_OK without a block.
- **V4L2Decoder: flag every input buffer as a frame end**: amvx maps V4L2_BUF_FLAG_KEYFRAME on the bitstream queue to end-of-frame. Without it VP8/VP9 buffers are rejected (QBUF -EIO) and H.264/HEVC input is treated as one byte stream, stamping several frames with one input's timestamp and stalling the pipeline.
- **DecodeComponent: drop extra output frames instead of failing**: A frame whose bitstream id no longer has a pending work (several pictures from one input) reported C2_CORRUPTED and killed the decode; log and drop it.
- **Decoder: support C2 low-latency mode**: Expose C2_PARAMKEY_LOW_LATENCY_MODE (MediaCodec KEY_LOW_LATENCY) and, when set, disable amvx frame reordering before the first STREAMON so pictures come out as soon as they are decoded instead of a DPB (~1 s at 12 fps) later. Used by the Miracast sink.
- **V4L2Encoder: handle SPS/PPS returned in a separate output buffer**: amvx returns the H.264 parameter sets in their own buffer with the first frame's timestamp; reporting it as a frame completed that work early and the IDR frame then failed with "Failed to find work". Cache them, prepend them to IDR frames and recycle the buffer.
- **V4L2Encoder: sync bitstream dma-bufs around CPU access**: Codec2 maps dma-bufs without DMA_BUF_IOCTL_SYNC. On non-coherent SoCs (SpacemiT K1) dirty cache lines from CPU writes were later written back over encoder output, corrupting random frames.
- **V4L2Encoder: merge VP8/VP9 frames split across output buffers**: amvx returns each VP8/VP9 frame in two buffers with the same timestamp (header/first partition without a frame type flag, then the rest). Accumulate the unflagged parts and emit them with the buffer that completes the frame; reporting the first part stalled the encoder with "Failed to find work".
- **V4L2Encoder: disable VP9 probability updates on amvx**: With explicit probability updates the amvx compressed header did not match the tile data and decoders rejected every inter frame. Devices without the private control ignore the request.

## Notes

- Verified on the BPI-F3: H.264/HEVC/VP8/VP9 decode, H.264/VP8 encode, VP9 encode up to 720p, Miracast low-latency decode.

## Build

```
./build.sh k1   # BPI-F3 (K1): kernel, bootloaders and Android images
./build.sh k3   # BPI-SM10 (K3), not booted yet
```

`./build.sh --help` lists the options (`--skip-kernel`, `--kernel-only`, ...).
