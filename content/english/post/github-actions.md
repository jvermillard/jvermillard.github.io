+++
author = "Julien Vermillard"
title = "Embedded software testing with GitHub Actions"
tags = [ "CI/CD", "Embedded", "Github"]
date = "2021-12-16"
+++

![Test Pyramid](/images/gha.png)

Continuous Integration pipelines can greatly improve your embedded software reliability and your release turn-over. But it’s a large investment and can be quite time consuming, especially for small teams.

Assembling and maintaining an “on target” test infrastructure can be hard to stabilize for several reasons.

![Test Pyramid](/images/test-pyramid.png)

My first advice, is to test as much as possible outside the target, unit test and integration testing have even more value than for non-embedded software.

Avoid on target testing as much as possible! It’s slow and difficult to keep stable over time!

![Test-Driven Development for embedded C](/images/tddfe.png)

I know that usage of unit testing is not that popular in embedded software, but I encourage you to try it! A good starting point is this book: [](https://amzn.to/30S1k0W)

This will greatly help you to reduce the amount of on target testing you need to do.

Another way to limit the number of on target test is to use a simulator, there is more and more options every year, for example, [Benjamin Cabé](https://twitter.com/kartben) made me discover [](https://renode.io/) which is quite useful for micro-controller targets

That said, you still need to run a handful of tests on your targeted device. At minimum, a couple of acceptance tests. And running this kind of test is looong because you need to reboot, flash, reboot, wait the system to be up, test, maybe retry because your network is flaky.

Furthermore, if you want to hook your test suite on your pull-request, you will need a pool of test bench to be able to parallelize the tests.

So, flaky test, multiplying test benches and having a master to control it can be, at minimum, a distraction up to a annoying daily burden for a software team. And in my experience, embedded software engineers are rarely comfortable with server infra technologies like Docker or K8S, server installation and monitoring. That’s why I started to look at moving from in-house maintained server/runner with managed solutions like GitHub Actions.

GitHub Actions let you run “action runners” on your own hardware in your own network. The runner communicate back with GitHub cloud servers using HTTP long polling, which minimize the number of holes you need to punch into your corporate network.

Action Runner are open source and are really easy to install on a Linux box, even on a Raspberry Pi: Go to the Actions setting in your repository and click the button to add a new Self Hosted Runner, just follow the instruction: download the package for the right architecture (X64/ARM/ARM64), decompress the archive and setup the systemd service using the provided ./svc.sh install script.

Remember to create a dedicated user and add it to the right groups to manipulate the needed I/O (for example, you will need the “dialout” group to access any serial port).

And that’s it! Your runner will appear on the GitHub action console!

For redirecting any job of your GitHub workflow to the host runner you need to specify it the .yml:

```yaml
runs-on:
— self-hosted
```

You can add more label to be able to target a specific runner connected to a specific variation of your hardware.

Now you have a working test bench, there is still something to tweak to stabilize your setup: in my experience, embedded system doesn’t like to be flashed multiple times per day, especially if you hook your CI on pull request commits.

After a couple of flashing my devices are randomly stuck, sometime due to some weird USB driver error, sometimes due to the previous test campaign which put the device in an unexpected state.

Anyway, it’s not a really common use case for most of the hardware platform to be flashed a dozen of time without a power cycle and it’s often complicated to correct it.

The workaround is quite simple: reboot and flash your device to a well know configuration before each test run. There are multiple ways to do this:

* a programmable power supply to shutdown your target before every test campaign, which can be costly and bulky but work (example: https://amzn.to/3jAjevM) use a USB driven relay to connected to your power supply (example:[](https://amzn.to/3GkzKd1))
![Reset using a relay](/images/relay.png)

* same trick but different: connect the reset signal to the relay and reset your device 
*if your device is powered over USB, maybe you don’t need any extra hardware! a lot of USB hub or USB controller power are software controllable: [](https://github.com/mvp/uhubctl),

Adopting GitHub action and private runners was really a breath. They are still quite new for me and I don’t have a lot of feedback at the moment, but until now the experience was pleasant and not having to deal with heavy weight and complicated CI servers was a relief.

I would be curious to hear about your experience, and your pain points when working with Continuous Integration and Embedded software testing. Let me know in the comments below!