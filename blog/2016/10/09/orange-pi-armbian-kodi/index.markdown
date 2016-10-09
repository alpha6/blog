---
tags: orange pi, kodi, armbian
title: How to fix sound issue with KODI on the orange pi
---

I have Orange Pi One board. It's a great product, it is faster, cheaper and smaller then Raspberry Pi, but it uses AllWinner H3 SOC and it is its main disadvantage.

If you want to use this board as a media center with KODI, you face the problem with sound. The sound works in Armbian applications but doesn't work in KODI. It happens because KODI uses S24_LE sampling format which is not supported by Orange Pi hardware.

  root@orangepione:~# speaker-test  -D hw:1 -c2 --format S24_LE

  speaker-test 1.0.28

  Format S24_LE is not supported...

Ok, we found out the root of the problem, time to fix it!

Now we need to set sampling format which is supported by Orange Pi hardware. One of supported sample formats is S32_LE.
Check if it is correct:

  root@orangepione:~# speaker-test  -D hw:1 -c2 --format S32_LE

  speaker-test 1.0.28

  Playback device is hw:1
  Stream parameters are 48000Hz, S32_LE, 2 channels
  Using 16 octaves of pink noise
  Rate set to 48000Hz (requested 48000Hz)
  Buffer size range from 64 to 131072
  Period size range from 32 to 16384
  Using max buffer size 131072
  Periods = 4

Now check soundcards, we need to get the device number which plays over HDMI output.

  root@orangepione:~# cat /proc/asound/cards
   0 [audiocodec     ]: audiocodec - audiocodec
                        audiocodec
   1 [sndhdmi        ]: sndhdmi - sndhdmi
                        sndhdmi

The `sndhdmi` is a soundcard which attracts our interest. Remember the number of this one.

Now we can edit the Alsa config:

  vim /etc/asound.conf

Delete old content and paste the following one:

  pcm.snd_card {
          type hw
          card 1
          device 0
  }

  ctl.snd_card {
          type hw
          card 1
          device 0
  }

  pcm.dmixer {
      type dmix
      ipc_key 1024
      ipc_perm 0666
      slave.pcm "snd_card"
      slave {
          period_time 0
          period_size 1024
          buffer_size 4096
          rate 48000
          format S32_LE
          channels 2
      }
      bindings {
          0 0
          1 1
      }
  }

Save the file and reboot.

After that the KODI will play sound.
If it doesn't play, check the number in `card` fields of the config, it must be the number from `sndhdmi` card, and samplerate and samplerate format in `slave` section. The samplerate should be equal with one from the `speaker-test` command output.
