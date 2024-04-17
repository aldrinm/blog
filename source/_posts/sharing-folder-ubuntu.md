---
title: Sharing a folder on ubuntu
date: "2024/02/25 00:00:00"
---



Preferably don't have your shared folders under home `~` because the default permissions on home has changed in recent Ubuntu versions and it is much harder to share them if they are located under home.

Create a folder on ubuntu under root `/`

{% codeblock line_number:false%}
mkdir /public-share```
{% endcodeblock %}


Change the owner of the folder (replace `aldrin` with your username)
{% codeblock line_number:false%}
chown -R aldrin:aldrin /public-share
{% endcodeblock %}


Install nautilus-share
{% codeblock line_number:false%}
apt install nautilus-share
{% endcodeblock %}


Update permissions for the `sambashare` folder
{% codeblock line_number:false%}
sudo usermod -aG sambashare aldrin
{% endcodeblock %}


In `Files`, find you folder, right-click and select the menu option `Sharing Options` (it is only available after installing nautilus-share)

- Tick the checkbox `Share this folder`
- Tick the checkbox `Allow others to access....`
- Tick the checkbox `Guest access...`

From the command line, find the ip address of the ubuntu device
{% codeblock line_number:false%}
hostname -I
{% endcodeblock %}


From the other device, let's say a Windows PC, type the IP address in the `Run` command window (windows key + R). For instance, `\\192.168.1.73`

