---
title:  "Ubuntu 20.04 on the Lenovo IdeaPad Gaming 3 15ARH05 (82EY006YUS)"
author: cldellow
tags:
  - ubuntu
  - linux
---

Every 4 years, I upgrade my laptop.

The good stuff: this often means a massive improvement in performance. For example, I just bought a Lenovo IdeaPad Gaming 3 15ARH05 (82EY006YUS). It's a big step up over my old MSI GL62 6QF. For $1,445 CAD ($1,241 for the laptop, $204 for the RAM upgrade), this is what I got:

<table>
<thead>
<tr>
<th></th><th>New</th><th>Old</th>
</tr>
</thead>
<tbody>
<tr>
<td>
<b>CPU</b><br/>
3x faster multicore<br/>
40% faster single core<br/>
(<a href="https://www.cpubenchmark.net/compare/AMD-Ryzen-7-4800H-vs-Intel-i7-6700HQ/3676vs2586">PassMark</a>)
</td>
<td>
<a href="https://www.amd.com/en/products/apu/amd-ryzen-7-4800h">AMD Ryzen 7 4800H</a>
</td>
<td>
<a href="https://ark.intel.com/content/www/us/en/ark/products/88967/intel-core-i7-6700hq-processor-6m-cache-up-to-3-50-ghz.html">Intel Core i7-6700HQ</a>
</td>
</tr>
<tr>
<td><B>RAM</b><br/>2.6x the size</td>
<td>8 GB 3200 MHz DDR 4 (stock)<br/>
2x16 GB 2666 MHz DDR 4 (upgraded)
</td>
<td>12 GB (8+4) 2133 MHz DDR 4 (stock)</td>
</tr>
<tr>
<td><B>SSD</b><br/>4x the size, <a href="https://ssd.userbenchmark.com/Compare/Toshiba-HG6-SATA-M2-128GB-vs-Pm991-NVMe-Samsung-512GB/m14053vsm931808">~2.2x</a> the speed</td>
<td>Samsung MZALQ512HALU
</td>
<td>Toshiba THNSNJ128G8NY</td>
</tr>
</tbody></table>

The bad stuff: installing Linux is usually a nightmare.

This time was not so bad, though! I installed Ubuntu 20.04. It recognized all of my hardware (wifi, ethernet, webcam, speakers, microphone)...but there was an issue with the touchpad. You could click, but you couldn't move the cursor.

This is <a href="https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1887190">a known issue affecting Lenovo IdeaPad touchpads on the 5.8 kernel</a>, which is the stock kernel on 20.04.

Happily, it's fixed in 5.11, which you can easily upgrade to by using <a href="https://github.com/bkw777/mainline">mainline</a>. You'll have to disable Secure Boot in your BIOS, otherwise you'll get an error during booting like:

<pre>
error: /boot/vmlinuz-5.11.0-051100-generic has invalid signature.
error: you need to load the kernel first.

Press any key to continue...
</pre>

Hopefully, this will eventually get backported to the 5.8.x kernel, and I can go back to using a distribution-supported kernel.

Other than this one hitch, things are working pretty smoothly! Kudos to Ubuntu and the Linux kernel maintainers!
