---
title: "Fix for Matlab Installer Bug on Fedora 33"
date: 2021-01-31
categories:
  - Linux
tags:
  - Linux
  - bug
  - matlab
  - quick
---

> TL;DR delete ``bin/glnxa64/libcrypto.so.1.1``
 
While installing [matlab](https://www.mathworks.com/products/matlab.html) on my Fedora 33 Workstation for my computer vision course ([CSCI 4831](https://experts.colorado.edu/display/coursename_207678)) I came across a weird bug with the matlab installer that made me miss Python immediately. After running the executable appropriately named ``install`` within the zip archive I was told to download, I received the following error:

```
terminate called after throwing an instance of 'std::runtime_error'
  what():  Unable to launch the MATLABWindow application
Aborted (core dumped)
```

After copying and pasting this into [ecosia](https://ecosia.org), I came across a discussion post (which can be found [here](https://www.mathworks.com/matlabcentral/answers/513449-what-unable-to-launch-the-matlabwindow-application-during-installation)), from someone who was trying to install the same version of matlab I was, but on [Manjaro](https://manjaro.org/). From the responses in the post, I learned that the error stems from trying to run the executable ``bin/glnxa64/MATLABWindow`` that is found within the installation archive.

I proceeded to run this executable directly and recieved the following error:

```
./bin/glnxa64/MATLABWindow: symbol lookup error: /lib64/libk5crypto.so.3: undefined symbol: EVP_KDF_ctrl, version OPENSSL_1_1_1b
```

After copying and pasting this into [ecosia](https://ecosia.org), I came across a bug report on RedHat's bugzilla (that can be found [here](https://bugzilla.redhat.com/show_bug.cgi?id=1829790)), which recommended to remove the file ``bin/glnxa64/libcrypto.so.1.1``. After doing this and executing the ``install`` executable once again, the matlab GUI installer popped right up without a hitch!

Here's a quote from **Petr** within the bug report who found the source of the issue:

> I learned something more about shared libraries and I finally found out where the problem is.
>
> Both products (Matlab & Scilab) contains their own shared library libcrypto.so.1.1, but uses system library libk5crypto.so.3.
> System library libk5crypto.so.3 uses new symbols, which system library libcrypto.so.1.1 has, but Matlab's library libcrypto.so.1.1 hasn't.
>
> So I deleted libcrypto.so.1.1 from Matlab directory and it works now (same as scilab).

## Bonus: Application Entry in Gnome

After working through the installer GUI and getting matlab all ready to go, you'll find that the installer doesn't add an application entry for GNOME. This means that the only way to actually run matlab is by executing ``matlab`` from the terminal. After some more searching on ecosia, here is a ``.desktop`` file that can be placed into ``~/.local/share/applications``:

```ini
[Desktop Entry]
Type=Application
Icon=matlab
Name=Matlab
Exec=/usr/local/bin/matlab -desktop -prefersoftwareopengl
Terminal=false
```

Be sure to restart GNOME in order for the application entry to appear, which can be done by logging out and logging back in, or by doing a quick ``ALT+F2, r, ENTER``.
