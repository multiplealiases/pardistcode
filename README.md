# pardistcode

Distributedly transcode a video file by splitting it into
segments, sending them to remote machines for transcoding,
and stitching the segments into an output file.

## Dependencies

These dependencies apply to the local machine and remote machines.

* Bash and GNU coreutils
  (you're free to figure out how to make this POSIX-compatible,
  but note the next dependency)

* GNU Parallel (`--sshlogin` requires that
  the remote machine(s) also have Parallel installed)

* rsync (the GNU Parallel args `--transfer` and `--return` need this)

* FFmpeg


## Help text

```
pardistcode: distributedly transcode a video file

Output is always in MKV.

Usage
pardistcode --input input.mkv \
    --sshlogin ':,2/user1@host1,1/user2@host2,1/user3@host3' \
    --ffmpeg-options '-c:v libsvtav1 -crf 38 -pix_fmt yuv420p10le -preset 4 -vf scale=-4:1080' \
    --audio-options '-c:a libopus -b:a 96k'

Options (defaults in [])
-h, --help            this help
-i, --input           input video file (required)
-o, --output          output video file without file extension [based on input]
-S, --sshlogin        --sshlogin arg to pass to GNU Parallel (required)
-f, --ffmpeg-options  transcode video with these options (required)
-a, --audio-options   transcode audio with these options [-c:a copy]
-l, --segment-length  split input video into segments of this length [30]
```

## Considerations

* x264 (`-c:v libx264`), x265 (`-c:v libx265`) and SVT-AV1 (`-c:v libsvtav1`)
  are very good at using every available core;
  start every `--sshlogin` entry with `1/` to only use 1 job on each machine.
  Whatever you do, always specify the job count,
  or it will end up running the remote computers'
  core count of jobs at once,
  which will probably cause an out-of-memory condition,
  in addition to the extreme lag.

* ...but your network might not be very good,
  which can cause noticable 'dead air' between jobs
  for faster encodes.
  To mitigate this, increase the number of jobs to 2 (`2/`)
  to make sure your remote computers always have something to do,
  or pass a higher `--segment-length` than 30 seconds.

* You should probably read `man parallel` for further info,
  (try searching for `jobs to remote computers`
  to land on the section for `--sshlogin`),
  but `:` in `--sshlogin` means "local machine".

* Your local machine should be able to SSH into your remote machines unattended.

* Segments default to being about 30 seconds long (respecting keyframes);
  this seems to provide a reasonable tradeoff between
  parallelism and encoding efficiency.

* Temporary files are stored in /tmp.
  To be safe, ensure that your /tmp is
  at least twice the size of the input file.

* Only video is distributed across remote machines.
  Audio is transcoded on the local machine
  (you get weird gaps/distortion on segment boundaries otherwise),
  subtitle and data streams are copied verbatim.

* You probably want to transcode multiple files.
  Try something like
  `parallel -q -j1 -0 pardistcode (...options...) -i {} :::: <(find . -type f -name '*.mkv' -print0)`

* I have `parallel` in this script configured to buffer output on job completion;
  you will not get visual feedback until the first segment finishes encoding.
  (consider monitoring your remote machines using `top`/`htop`/`btop`/`glances`/whatever)

* Output is always in MKV. If you need it in MP4/MOV, do
  `ffmpeg -i in.mkv -map 0 -c:a copy -c:v copy -c:s copy -c:d copy -movflags faststart out.mp4`
  afterwards.

