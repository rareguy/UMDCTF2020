# UMDCTF 2020

## Jarred 3
So I only intended to write a writeup for this challenge only because I really love this for an introduction to memory forensics. I might not get this chance twice, so I was intrigued to try it.

> Jarred is always having issues. He thinks he got malware from doing something dumb, but won't tell me what he was doing?

> no real malware to worry about here

> Drive link: https://drive.google.com/open?id=1g0j7rm53lU7ZRBM8XgkPkFpUgsfXpjfi

That link might or might not be invalid by the time you reached this writeup, that depends on the author, drkmrin78.

So we were given a quite big vmem file in a compressed tar.gz format. Based on Jarred-1 (which solution is in the same area with Jarred-3), we can use [Volatility](https://github.com/volatilityfoundation/volatility), an advanced memory forensics tool. This tool helps us to parse memory stuffs inside a vmem memory based on OS architecture preset which are provided in the repository.

I cloned it, and use it on the vmem file, with Win7SP1x64 preset. I can use Volatility out of the box without installing via pip so that's a cool feature.

*I renamed the file to jarred3.vm to ease myself*
First thing I do is look at the process running when the memory was captured.
`python vol.py -f ../jarred3.vmem --profile=Win7SP1x64 pslist`
![pslist](https://github.com/rareguy/UMDCTF2020/blob/master/pslist.png?raw=true)

There were lots of Windows services, and some browsers, a Thunderbird email client, and some weird looking helper.exe. This was my first suspicion. (spoiler: wrong)

Then, I tinkered a bit on the vmem, doing `dlllist`, `malfind`, `filescan`, nothing too interesting. I spent like MBytes of output of those commands, filtering it out, and nothing unusual there.

And then I think, hey, the description tells us about malware, so there might be something in the running process itself. Moreover he seemed like he used tor to browse.

So I carve all of the process, which surprisingly quick. But then...

`python vol.py -f ../jarred3.vmem --profile=Win7SP1x64 procdump --dump-dir ../result`

![virus!](https://github.com/rareguy/UMDCTF2020/blob/master/Inkedvirus_LI.jpg?raw=true)

When Volatility extracted it all, there was one executable file, yes, only one, that were caught by Avast as a literal malware. And then I asked the author about the existence of this virus, which he said there was no real malware. He said something like _"don't worry, it's not a real malware. Windows might not like it. you're on the right track."_ that last bit clicked something in my mind that this process contains the malware we're looking for. So I dumped the memory which used by the process, which PID is `424` as stated in the Avast malware warning. It was the `thunderbird.exe` all along.

`python vol.py -f ../jarred3.vmem --profile=Win7SP1x64 memdump -p 424 --dump-dir ../result`

I thought this was closer to the answer, but it's not, really. From the dump we got 400+MB of Thunderbird's memory dump.
![bruh](https://github.com/rareguy/UMDCTF2020/blob/master/424%20dump.png?raw=true)

I worked with a friend to filter this out for an hour while also working on other challenges. We concluded that there are some email warnings of _potentionally dangerous attachment_ which was a good sign that we're heading on the right direction.

`strings 424.dmp | grep virus`

![warnings](https://github.com/rareguy/UMDCTF2020/blob/master/warning.png?raw=true)

And then my friend found an enlightenment, from a command to see the output around the result.

`strings 424.dmp --radix=d | grep -A 300 jarred`

We ran that command and scrolled up a bit to find a capture of email attachment request somewhat similar to packet capture.
![attachment](https://github.com/rareguy/UMDCTF2020/blob/master/attachment.png?raw=true)

And then, I ran HexEditor to copy that base64 from around line 420,000,000, put it in TXT, clean and decode it through `python` script to get a PKZIP file. Extracted it, got a DOCX file. The DOCX file is `LovinInvoice.docx`

Usually MS Office-based virus has the code on the Macros section. And because this is a DOCX, which basically a spicy zip containing XMLs, we can open the macros without the hassle of risking our computer.

![vba is not visual boy advance](https://github.com/rareguy/UMDCTF2020/blob/master/vba.png?raw=true)

We found the VBA macro inside the DOCX, which unsurprisingly contained malware and alerted Avast to do that _ding ding_ sound, and quarantined the file, just like us (haha get it because at the time of this writing we were all in quarantine).

VBA macros are already compiled for the DOCX, so we need to decompile it with a python toolkit for deconstructing DOCX file, [oletools](https://github.com/decalage2/oletools).

Installed it with `pip`, ran `olevba vbaProject.bin --decode` and we will get our decompile result.
![decompile](https://github.com/rareguy/UMDCTF2020/blob/master/decompile.png?raw=true)

There was a keyword that looks like a flag in reverse.
Reversed it.

`-{mJ_4_e3e0Jq_x0J3fk1u5_3ph3jl!!!}`

Do a caesar cipher +18, add the format of the flag, and we got our flag.

`UMDCTF-{uR_4_m3m0Ry_f0R3ns1c5_3xp3rt!!!}`
