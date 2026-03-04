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

### Network latency: segment length and job count

**Always** pass the number of jobs to run in `--sshlogin`,
or you will find your machines OOMing and lagging heavily
because they're running `$(nproc)` jobs.

It is very likely that x264 (`-c:v libx264`),
x265 (`-c:v libx265`) and SVT-AV1 (`-c:v libsvtav1`)
can parallelize to every available core on your machines,
so assuming a fast network, you could probably get away with specifying `1/`
(run only 1 job at a time) for every remote machine.

If your network isn't fast, you might notice 'dead air' between jobs.
There are 2 ways to mitigate this:

* Do `2/` (2 jobs) instead. This cuts out most of the dead air,
  but obviously running 2 FFmpegs means you're using twice the RAM,
  and it might cause intense lag
  if your machines are being used interactively.

* Use longer values of `--segment-length` to make
  the dead air proportionally smaller.
  Increasing it too far might decrease the degree of parallelism you can attain
  and delay transcodes if you have a mix of fast and slow machines.

### Unsorted

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

