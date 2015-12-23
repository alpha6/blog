---
tags: Spotlight, ElCapitan, MacOS
title: Spotlight does not search for
---

If your Spotlight does not search for some files or applications:

The first of all - restart the Spotlight:

    sudo launchctl unload -w /System/Library/LaunchDaemons/com.apple.metadata.mds.plist
    sudo launchctl load -w /System/Library/LaunchDaemons/com.apple.metadata.mds.plist

If it have no effect - remove Spotlight's index:

Stop Spotlight service:

    sudo launchctl unload -w /System/Library/LaunchDaemons/com.apple.metadata.mds.plist

Remove Spotlight index folder:

    sudo rm -rf /.Spotlight-V100

Run service:

   sudo launchctl load -w /System/Library/LaunchDaemons/com.apple.metadata.mds.plist

After reindex - everything should work well.
