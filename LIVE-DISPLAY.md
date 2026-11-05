# Live Display build of libva-v4l2_request

The `live-display` branch is [megi's VA-API driver](https://xff.cz/git/libva-v4l2_request) plus six patches. They let Chromium and Electron decode HEVC with the Raspberry Pi HEVC decoder (`rpi-hevc-dec`), with no copy between decoder and GPU.

Live Display uses it in the player wrapper of its Raspberry Pi 4 boxes. The Pi 4 H.264 decoder is stateful: this driver does not drive it, so only HEVC is accelerated.

## Branches and tags

| Ref | Content | Rule |
|---|---|---|
| `master` | Exact mirror of megi's `master` | Only fast-forwarded from `upstream`, never committed to |
| `live-display` | `master` at a tested commit + the Live Display patches | Rebased onto each new upstream release |
| `ld-N` tags | The builds the boxes install | Set after a box test, never moved |

The patches are never merged into `master`: rebasing keeps them as a short, readable series on top of megi's code, ready to be proposed upstream.

## Patches

| Commit | Problem it fixes |
|---|---|
| Do not require a converter for tiled CAPTURE formats | `no usable format converter for Nc12`: hardware decoding refused |
| Queue multiplanar CAPTURE buffers with all their planes | `failed to queue CAPTURE buffer: Invalid argument` (2-plane `Nc12`) |
| Fall back to Broadcom SAND for the backing of unbound surfaces | `vaExportSurfaceHandle failed` in Chromium |
| Export Broadcom SAND surfaces as separate layers | Chromium refuses the frame (`kFailedToCreateVideoFrame`) |
| Decode into the surface backing through imported CAPTURE buffers | Pink and green picture: Chromium showed memory the decoder never wrote |
| Finish the decode of frames without references before returning | Orange and green flash when a video loops or seeks |

## Measured results

Same 1080p30 video on a Raspberry Pi 4, Electron 43 (Chromium 150):

| Decoding | CPU (4 cores = 400 %) | Dropped frames |
|---|---|---|
| H.264, software (FFmpeg) | about 212 % | 10 to 17 % |
| HEVC, this driver | about 24 % | about 1 % |

## Build and install on a box

1. Install the build dependencies:

   ```bash
   sudo apt install -y git meson ninja-build pkg-config gcc libva-dev libdrm-dev vainfo && echo "OK: dependencies installed"
   ```

2. Clone the tag to install (`ld-1` here):

   ```bash
   git clone --branch ld-1 https://github.com/live-display/libva-v4l2_request.git ~/libva-v4l2_request && echo "OK: driver cloned"
   ```

3. Configure the build:

   ```bash
   meson setup ~/libva-v4l2_request/build ~/libva-v4l2_request && echo "OK: build configured"
   ```

4. Compile:

   ```bash
   ninja -C ~/libva-v4l2_request/build && echo "OK: driver built"
   ```

5. Install:

   ```bash
   sudo ninja -C ~/libva-v4l2_request/build install && echo "OK: driver installed"
   ```

6. Check that the driver lists HEVC:

   ```bash
   sudo env LIBVA_DRIVER_NAME=v4l2_request vainfo 2>/dev/null | grep -q VAProfileHEVCMain && echo "OK: HEVC available" || echo "FAIL: HEVC not listed"
   ```

Chromium then needs `LIBVA_DRIVER_NAME=v4l2_request` in its environment and `--enable-features=VaapiIgnoreDriverChecks,PlatformHEVCDecoderSupport` on its command line.

## Update to a new upstream release

Run on a computer, in the clone that has `upstream` (megi) and `origin` (this repository) as remotes.

1. Fetch megi's changes:

   ```bash
   git -C ~/libva-v4l2_request fetch upstream --tags && echo "OK: upstream fetched"
   ```

2. Move to `master`:

   ```bash
   git -C ~/libva-v4l2_request switch master && echo "OK: on master"
   ```

3. Fast-forward the mirror (fails if `master` ever diverged, which must not happen):

   ```bash
   git -C ~/libva-v4l2_request merge --ff-only upstream/master && echo "OK: master updated"
   ```

4. Publish the mirror:

   ```bash
   git -C ~/libva-v4l2_request push origin master --tags && echo "OK: master pushed"
   ```

5. Move to `live-display`:

   ```bash
   git -C ~/libva-v4l2_request switch live-display && echo "OK: on live-display"
   ```

6. Replay the patches onto the new release (see [If something goes wrong](#if-something-goes-wrong) on a conflict):

   ```bash
   git -C ~/libva-v4l2_request rebase master && echo "OK: patches replayed"
   ```

7. Publish the rewritten branch (released builds stay reachable through their `ld-N` tags):

   ```bash
   git -C ~/libva-v4l2_request push --force-with-lease origin live-display && echo "OK: live-display pushed"
   ```

8. Build the branch on a test box ([Build and install on a box](#build-and-install-on-a-box), cloning `--branch live-display`) and check the playback:
   - the journal of the player shows `using CAPTURE format Nc12 (…), imported buffers`;
   - the DevTools Media panel shows `VaapiVideoDecoder` with `Hardware decoder: true`;
   - a looping HEVC video plays with a correct picture and no flash when it restarts;
   - `video.getVideoPlaybackQuality()` drops about 1 % of the frames.

9. Tag the tested build, with the next number:

   ```bash
   git -C ~/libva-v4l2_request tag -a ld-2 -m "Live Display build on upstream <version>" live-display && echo "OK: tag set"
   ```

10. Publish the tag:

    ```bash
    git -C ~/libva-v4l2_request push origin ld-2 && echo "OK: tag pushed"
    ```

## If something goes wrong

| Symptom | Fix |
|---|---|
| `rebase` stops on a conflict | megi changed the code a patch touches. Edit the file, then `git add` it and `git rebase --continue`. To give up: `git rebase --abort` |
| `merge --ff-only` fails | Someone committed to `master`. Move those commits to `live-display`, then reset `master` to `upstream/master` |
| `vainfo` lists HEVC but Chromium shows a black or pink picture | The headers lacked the Raspberry Pi formats, so their code was left out. Build on Raspberry Pi OS, whose `linux-libc-dev` defines `V4L2_PIX_FMT_NV12MT_COL128` |
| Chromium falls back to software (`FFmpegVideoDecoder`) | `LIBVA_DRIVER_NAME` or the features above are missing from the Chromium process |
