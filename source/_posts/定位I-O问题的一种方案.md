---
title: 定位I/O问题的一种方案
date: 2017-07-22 18:45:59
tags:
---
# Monitor disk io on linux server with iotop and cron
Recently my server was giving notifications of disk io activity rising above a certain threshold at regular intervals. My first guess was that some cronjob task was causing that. So I tried to check various cron tasks to find out which task or process was causing the io burst. On servers specially its always a good practice to monitor resource usage to make sure that websites work fast and well.

## Automatic logging via cron
Cron will run iotop in the background and record io usage details to a file that can be analysed later.

Here is the basic iotop command that we want to run in the background via cron.
`* * * * * root /usr/sbin/iotop -botqqqk --iter=60 | grep -P "\d\d\d\d\.\d\d K/s"  >> /var/log/iotop`

Now the most important option used in the above command is the "b" option which is for batch/non-interactive mode. In batch mode iotop will keep outputting line after line instead of showing a long list that updates automatically. This is necessary when we want to log io activity over a certain period of time.

The other option called "o" will show only those processes which actually did some io activity. Otherwise iotop would show all processes. The t option adds a timestamp which adds further information if you want to track a specific high io process. The k option shows all figures in kilobytes.

To log the output we simply need to redirect it to a file. The best place is /var/log and the file could be named iotop. So here is the next command
` >> /var/log/iotop`

## Monitor only high io processes
Filtering result by applying `grep -P "\d\d\d\d\.\d\d K/s`,thus it would not show those process that had less than 1000 K/s of disk io.
