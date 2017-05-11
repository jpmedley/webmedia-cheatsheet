# Web Media Cheat Sheet DRAFT

This document shows in order commands needed to get from a raw mov file to
encrypted assets packaged for DASH or HLS. For the sake of having a goal to
illustrate, I'm converting my source file to a bit rate of 8Mbs at a resolution
of 1080p (1920 x 1980). Adjust these values as your needs dictate.
 
Conversion is done with two applications:
[Shaka Packager](https://github.com/google/shaka-packager) and
[ffmpeg](https://ffmpeg.org/download.html). Although
I've tried to show equivalent operations for all procedures, not all operations
are possible in both applications.
 
In many cases, commands may be combined in a single command line operation, and
in actual use would be. For example, there's nothing preventing you from setting
an output file's bit rate in the same operation as a file conversion. For this
cheat sheet, I will show these operations (among others) as separate commands
for the sake of clarity.
 
Please let me know of useful addition or corrections. Pull requests are welcome.

## Display media file characteristics

    packager input=myvideo.mp4 --dump_stream_info

    ffmpeg -i myvideo.mp4

This is technically an incorrect usage of ffmpeg. As such it prints an error
message to the screen. It does however list information not available from the
Shaka Packager command.

## Demuxing (splitting audio and video)

My preference would be to show file type conversion before diving into demuxing.
It appears that Shaka Packager cannot do conversion without also demuxing.

**mp4/Shaka Packager**
 
    packager input=myvideo.mp4,stream=video,output=myvideo_video.mp4
    packager input=myvideo.mp4,stream=audio,output=myvideo_audio.m4a
 
With Shaka Packager you can combine these.
 
    packager \
      input=myvideo.mp4,stream=video,output=myvideo_video.mp4 \
      input=myvideo.mp4,stream=audio,output=myvideo_audio.m4a
      
**webm/Shaka Packager**

    packager \
      input=myvideo.webm,stream=video,output=myvideo_video.webm \
      input=myvideo.webm,stream=audio,output=myvideo_audio.webm
 
**mp4/ffmpeg**
 
    ffmpeg -i myvideo.mp4 -vcodec copy -an myvideo_video.mp4
    ffmpeg -i myvideo.mp4 -acodec copy -vn myvideo_audio.m4a
    
**webm/ffmpeg**
 
    ffmpeg -i myvideo.webm -vcodec copy -an myvideo_video.webm
    ffmpeg -i myvideo.webm -acodec copy -vn myvideo_audio.webm

## Convert files types

Shaka Packager cannot process mov files and hence cannot be used to convert
files from that format.  

### mov to mp4
 
    ffmpeg -i myvideo.mov myvideo.mp4

### mov to webm

When converting a file to webm, ffmpeg doesn't provide the correct aspect
ration. Fix this with a filter (`-vf setsar=1:1`).
 
    ffmpeg -i myvideo.mov -vf setsar=1:1 myvideo.webm

## Set the bit rate

For ffmpeg, I can do this while I'm converting to mp4 or webm.
 
    ffmpeg -i myvideo.mov -b:v 8M myvideo.mp4
    ffmpeg -i myvideo.mov -vf setsar=1:1 -b:v 8M myvideo.webm
 
For Shaka Packager:
 
    packager \
      input=myvideo.mp4,stream=audio,output=myvideo_audio.mp4 \
      input=myvideo.mp4,stream=video,output=myvideo_video.mp4,bandwidth=8000000

## Make sure audio and video synchronize during playback

To ensure that audio and video synchronize during playback insert keyframes.
 
    ffmpeg -i myvideo.mp4 -keyint_min 150 -g 150 -f webm -vf setsar=1:1 out.webm
 
    packager \
      ??

## Change the Codec

You might have an older file whose codec you want to update.

### mp4/H.264
 
    ffmpeg -i myvideo.mp4 -c:v libx264 -c:a copy myvideo.mp4

### Audio for an mp4

    ffmpeg -i myvideo.mp4 -c:v copy -c:a aac myvideo.m4a

### webm/VP9

    ffmpeg -i myvideo.webm -v:c libvpx-vp9 -v:a copy myvideo.webm

### Audio for a webm

    ffmpeg -i myvideo.webm -v:c copy -v:a libvorbis myvideo.webm
    ffmpeg -i myvideo.webm -v:c copy -v:a libopus myvideo.webm

## Create a DASH MPD file

    packager \
      input=myvideo.mp4,stream=audio,output=myvideo_audio.mp4 \
      input=myvideo.mp4,stream=video,output=myvideo_video.mp4 \
      --mpd_output myvideo_vod.mpd

## Create HLS resources

    ffmpeg -i myvideo.mp4 -c:a copy -b:v 8M -c:v copy -f hls -hls_time 10 \
           -hls_list_size 0 myvideo.m3u8

## Clear Key Encryption

### Create a key

The same method for creating a key may be used with both DASH and HLS. The
following will create a key made of 16 hex values.
 
    openssl rand 16 > media.key

### Encrypt for DASH

For the `-key` flag can be the key created earlier and stored in the media.key
file. However, when entering it on the command line, be sure to remove the
whitespace from the key. For the `-key_id` flag repeat the key value.

> **Note:** Technically, the key ID is supposed to be either the first 8 OR the
> first 16 hex digits of the key. Since packager requires the key to be 16
> digits and does not allow a 32 digit key, both flags use the same value.

 
    packager \
      input=myvideo.mp4,stream=audio,output=glocka.m4a \
      input=myvideo.mp4,stream=video,output=glockv.mp4 \
      --enable_fixed_key_encryption --enable_fixed_key_decryption \
      -key 7bbef28644876a545a40caaaad9b3200 -key_id 7bbef28644876a545a40caaaad9b3200 \

### Create a key information file

To encrypt for HLS you need a key information file in addition to a key file. A
key information file has the following format.
 
    key URI
    key file path
 
For example:
 
    https://example.com/media.key
    media.key

### Encrypt for HLS

    packager \
      'input=input.mp4,stream=video,segment_template=output$Number$.ts,playlist_name=video_playlist.m3u8' \
      'input=input.mp4,stream=audio,segment_template=output_audio$Number$.ts,playlist_name=audio_playlist.m3u8,hls_group_id=audio,hls_name=ENGLISH' \
      --hls_master_playlist_output="master_playlist.m3u8" \
      --hls_base_url="http://localhost:1000/"
 
This command will accept a key with either 16 or 32 characters.
 
    ffmpeg -i myvideo.mov -c:v libx264 -c:a aac -hls_key_info_file key_info myvideo.m3u8

## Widevine Encryption

Everything in this command except the name of your files and the `--content-id`
flag should be copied exactly from the example. The `--content-id` is 16 or 32
random hex digits.
 
    packager \
      input=glocken.mp4,stream=video,output=enc_glocken.mp4 \
      input=myvide.mp4,stream=audio,output=enc_myaudio.m4a \
      --enable_widevine_encryption \
      --key_server_url "https://license.uat.widevine.com/cenc/getcontentkey/widevine_test" \
      --content_id "fd385d9f9a14bb09" --signer "widevine_test" \
      --aes_signing_key "1ae8ccd0e7985cc0b6203a55855a1034afc252980e970ca90e5202689f947ab9" \
      --aes_signing_iv "d58ce954203b7c9a9a9d467f59839249"

## All Together Now

### Packager/DASH

### Packager/HLS

### ffmpeg/HLS

