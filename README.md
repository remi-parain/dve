# dve - the distributed video encoder

This is a small script to do distributed, high quality video encoding.

The script:

- breaks input video into chunks.
- distributes chunks to different servers via SSH.
- encodes those chunks in parallel.
- reassembles the chunks into final encoded video.

Why do this? So you can encode video using the best settings possible,
and use as many machines as you have available to ensure it doesn't
take forever. ☺

## Usage

By default, dve will just use your local host for encoding, which isn't likely to
improve performance. At a bare minimum, you should specify more than one host
to encode with:

    dve -l host1,host2,host3 media/test.mp4

After the encoding is completed and the chunks stitched back together, you
should end up with an output file named something like "original_new.mkv" in
your current working directory. You can adjust output naming, but note that the
output container format will currently always be mkv:

    dve -s .encoded.mkv -e ~/bin/ffmpeg -l host1,host2,host3 media/test.mp4

Encoding currently breaks input videos into 1m (60s) chunks. This should give
reasonable parallelism across a reasonable number of hosts. If you have
many hosts you may need to adjust this down using -t. If you have a small number
of hosts and a long video, you may wish to bump this up to encode larger chunks
and get marginally better compression. Values larger than 300 (15m) are
probably a waste of time.

Since the ffmpeg situation in Ubuntu has been resolved, dve no longer
tries to copy over your local copy of ffmpeg for encoding, which greatly
simplifies the script logic. This means you need to have an ffmpeg binary on
every system used for encoding, and if you specify a custom path, that custom
path should be the same on every system.

## Benchmarks

Hosts used for this benchmark were dual Xeon L5520 systems with 24GB of RAM,
16 HT cores per host. Input video file is a 4k resolution (4096x2304) test
clip, 3:47 in length.

### ffmpeg on a single host

```bash
$ time nice -n 10 ./ffmpeg -y -v error -stats -i test.mp4 -c:v libx264 -crf 20.0 -preset medium -c:a libvorbis -aq 5 -f matroska test.mkv
frame= 5459 fps=7.4 q=-1.0 Lsize=  530036kB time=00:03:47.43 bitrate=19091.2kbits/s
real    12m17.177s
user    182m57.340s
sys     0m36.240s
```

### dve with 3 hosts

```bash
$ time dve -o "-c:v libx264 -crf 20.0 -preset medium -c:a libvorbis -aq 5" -l c1,c2,c3 test.mp4
Creating chunks to encode

Computers / CPU cores / Max jobs to run
1:local / 2 / 1

Computer:jobs running/jobs completed/%of started jobs/Average seconds to complete
ETA: 1s 1left 1.57avg  local:1/7/100%/1.6s
Running parallel encoding jobs

Computers / CPU cores / Max jobs to run
1:c1 / 16 / 1
2:c2 / 16 / 1
3:c3 / 16 / 1

Computer:jobs running/jobs completed/%of started jobs/Average seconds to complete
ETA: 380s 6left 64.00avg  c1:1/1/40%/132.0s  c2:1/0/20%/0.0s  c3:1/1/40%/132.0s
Computer:jobs running/jobs completed/%of started jobs
ETA: 90s 2left 45.33avg  1:1/2/37%/138.0s  2:0/2/25%/138.0s  3:1/2/37%/138.0s
Computer:jobs running/jobs completed/%of started jobs/Average seconds to complete
ETA: 42s 1left 42.14avg  c1:0/3/37%/99.7s  c2:0/2/25%/149.5s  c3:1/2/37%/149.5s
Computer:jobs running/jobs completed/%of started jobs
ETA: 50s 1left 50.29avg  1:0/3/37%/118.7s  2:0/2/25%/178.0s  3:1/2/37%/178.0s
Combining chunks into final video file
Cleaning up temporary working files

real    6m17.075s
user    1m29.630s
sys     0m22.697s
```

### Summary

dve has overhead, due to breaking the source file into chunks, transferring those chunks
across the network, retrieving the encoded chunks, and recombining into a new file.

Given these limitations, a ~2x speed increase by using 3 encoding machines is
a reasonable improvement over using a single system.

If you've got benchmarks using more hosts, please submit them!

## Installation

### SSH

SSH is used by GNU parallel to distribute the jobs to target systems.
It's recommended that you use "ssh-keygen" and "ssh-copy-id" to
setup key based authentication to all your remote hosts.

### Pre-reqs

The following need to be installed on the host running this script:

- [ffmpeg](https://www.ffmpeg.org/download.html)
- [GNU parallel](https://www.gnu.org/software/parallel/)

It's recommended that you use recent (>= 2.5.x) versions of ffmpeg to ensure
they have all the required functionality for splitting and combining the
video chunks.

### Windows

dve can be run on Windows via [cygwin](http://www.cygwin.com/).

To do so, you'll need to:

- build GNU parallel manually from source (requires make).
- install a static build of [ffmpeg for Windows](http://ffmpeg.zeranoe.com/builds/).
- install (or symlink) above into your $PATH, usually ~/bin.

You'll also need to do the following if you want to use the host to render with:

- [configure sshd](http://www.noah.org/ssh/cygwin-sshd.html)
- alter ~/.bashrc as mentioned above.

## Restrictions

- currently only generates mkv containers on output.

## ⚠ Known Issues

See the [GitHub issues page](https://github.com/nergdron/dve/issues)

## License
dve is copyright 2013-2015 by Graeme Nordgren <graeme@sudo.ca>.

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see the
[GNU licenses page](http://www.gnu.org/licenses/).
