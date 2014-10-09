# test-engine-live-tools

Small tools and scripts for the EBU test engine platform. These allow for DASH-ing and encoding a live stream.

# Installation

Clone the code and install the dependencies using npm. Be sure to provide your own MP4Box and ffmpeg binaries, they are not
included.

    # git clone https://github.com/ebu/test-engine-live-tools.git
    # cd test-engine-live-tools
    # npm install

In the future this package will be provided as a npm package.


# Usage

In general, the tool operates by looping an MPEG-2 Transport Stream and providing that to ffmpeg using the segment muxer.
The generated segments are picked up and packaged/fragmented using MP4Box. After all segments for the desired representations
are available they are DASH-ed by MP4Box using the `--dash-ctx` option. This generates the MPD for the live stream and keeps
it updated.

A command-line utility is included which allows you to easily stream from the command-line. Usage:

    # bin/live-stream [-c config_file] input_file

The `config_file` is optional and allows you to create custom settings for your live stream without having to change anything
in the sources. The `input_file` is mandatory and it is required that this is currently muxed as a MPEG-2 Transport Stream for
easy looping of the source material. The contents of the file can be either MPEG-2 video/audio or H264/AAC, or most likely
anything else that ffmpeg can extract from a MPEG-2 TS container.

# Configuration

The default configuration generates one video representation and one audio representation. See `lib/config.js` for details.
All parameter configuration is listed as an array of arguments which NodeJS understands. Custom configuration can be created
by creating a valid JSON file containing an object that overrides values of the default configuration. The complete default configuration is:

```javascript
{
  segmentDir: '/tmp/dash_segment_input_dir',
  outputDir: '/tmp/dash_output_dir',
  mp4box: 'MP4Box',
  ffmpeg: 'ffmpeg',
  encoding: {
    commandPrefix: [ '-re', '-i', '-', '-threads', '0', '-y' ],
    representations: {
      audio: [
        '-map', '0:1', '-vn', '-acodec', 'aac', '-strict', '-2', '-ar', '48000', '-ac', '2',
        '-f', 'segment', '-segment_time', '4', '-segment_format', 'mpegts'
      ],
      video: [
        '-map', '0:0', '-vcodec', 'libx264', '-vprofile', 'baseline', '-preset', 'veryfast',
        '-s', '640x360', '-vb', '512k', '-bufsize', '1024k', '-maxrate', '512k',
        '-level', '31', '-keyint_min', '25', '-g', '25', '-sc_threshold', '0', '-an',
        '-bsf', 'h264_mp4toannexb', '-flags', '-global_header',
        '-f', 'segment', '-segment_time', '4', '-segment_format', 'mpegts'
      ]
    }
  },
  packaging: {
    mp4box_opts: [
      '-dash-ctx', '/tmp/dash_output_dir/dash-live.txt', '-dash', '4000', '-rap', '-ast-offset', '12',
      '-no-frags-default', '-bs-switching', 'no', '-min-buffer', '4000', '-url-template', '-time-shift',
      '1800', '-mpd-title', 'MPEG-DASH live stream', '-mpd-info-url', 'http://ebu.io/', '-segment-name',
      'live_$RepresentationID$_', '-out', '/tmp/dash_output_dir/live', '-dynamic', '-subsegs-per-sidx', '-1'
    ]
  }
}
```

You can use this as a basis for your own configuration. Both the ffmpeg command and MP4Box commands are generated
using this configuration. You can add extra representations by overriding the `encoding` object.

Please not that some options are required to be able to create a valid live stream. For example: it is required that
we read from stdin using ffmpeg, so "-i -" is required. Also it is preferred to read realtime, so "-re" is also needed.

More example configuration will be added add a later stage.

# Caveats / pitfalls

Live streaming using MPEG-DASH is not always easy. To make sure that you are compatible with most DASH clients take extra
care to make sure you're always generating closed GOPs, fixed segment durations and constant bit rates and that timing
over segments is continuous and identical across representations. The default configuration should do just that, but be
sure to keep this in mind.

# Requirements

* NodeJS 0.10.x / npm
* ffmpeg binary compiled to taste (recommended: 2.3 or higher)
* MP4Box binary compiled to taste (recommended: r5400 or newer)

# License

Available under the BSD 3-clause license.
