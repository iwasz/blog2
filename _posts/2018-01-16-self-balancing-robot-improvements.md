---
layout: post
title: Self balancing robot improvements
permalink: /uncategorized/self-balancing-robot-improvements/
post_id: 509
categories: 
- Uncategorized
---
I was dissatisfied with robot's performance, so I experimented a lot with various aspects to improve it. But along the way, the robot, however simple in concept, proved to be a pretty complex system which depends on many factors that I didn't anticipate. This variety of factors which altogether influence its movements make it very difficult to troubleshoot problems. So, now I'll try to recall my adventures with it in order. Oh, and my objective (best case scenario) is to make it stable to the point it stands completely still. This is what I found the most difficult.

[![Ballancing robot ballances](https://img.youtube.com/vi/M3uyGxLnFOw/0.jpg)](http://www.youtube.com/watch?v=M3uyGxLnFOw)

### Better motors

![Better motors picture](/assets/IMG_20180107_163408-e1516133185673-169x300.jpg)

I decided, my first set of motors (Dagu DG02S 48:1) have to low torque on low RPM (driven from DRV8835 H-Bridge again with PWM). The robot was able to stand, but wiggled (video in the previous post). So I changed them to JK28HS32 stepper motors, which proved to be even weaker. They simply have to low torque in all RPM range for the weight of my robot. More disadvantages:

* Draws tons of current even when robot is stationary. This is because un-geared steppers (at least small ones) spin freely when unpowered and thus need current which hold them in position.
* More complex electronics (1 H-bridge per motor instead of 1/2 per brushed one), and much more complex programming.	
* Intense vibration even in half stepping mode. And I used 200step ones. The solution for that would be to use smaller wheels with soft tires or implement micro stepping which would in turn reduce its 
[incremental torque](https://www.micromo.com/technical-library/stepper-motor-tutorials/microstepping-myths-and-realities). Every time I use one of these steppers, and I dig into the subject I realize how difficult a stupid motor might get.
So, not without disappointment I moved to another set of motors which happened to be 
[Brushless DC Motor with Encoder 12V 159RPM](https://www.dfrobot.com/index.php?route=product/product&product_id=1364&search=FIT0441&description=true#.VnlABPnhBUR) from DFrobot (I found them on Aliexpress BTW). To my consternation it didn't help much. Advantages of the newly mounted motors are its high torque, ease of interfacing (they have controller stuffed inside), but cons are:
* They have quite some backlash. 	
* They run on 12V, so I had to rework some of the electronics. 	
* They might be a little bit faster, though at the end I was able to stabilize the thing pretty satisfactory.

![Motors ina a chassis](/assets/IMG_20180107_173324-300x205.jpg)

To sum up : high-quality motors eliminate one point of failure and let you focus on different aspects if something is failing. My ideal motors, that I would like to have are:

* Precise in terms of control. Wide range of RPMs i.e. able to spin super slow or quite fast with decent torque in all situations. What is a decant torque? Dunno. 2,4 kg*cm?
* Fast. I would love to have 300RPM.	
* Minimal backlash. 	
* Encoders built-in. 	
* Not to mention power efficiency and ease of programming, but this is not the most important thing.

### Surface
So then I finally decided, that I stop there with changing motors, and won't replace them for the 4th time, but try to tune the PID and fiddle with the software. I replaced my fusion algorithm from simple complementary one to Madgwick, which open-source implementation is 
[available online](http://x-io.co.uk/open-source-imu-and-ahrs-algorithms/), and it is a part of a PhD thesis of some clever guy (Sebastian Madgwick). I cannot stress how far superior this algorithm is over my humble thing. But no luck with that neither. If I increased Kp and Ki it would oscillate vigorously (Kd was of not  much help there), and when I decreased Kp and especially Ki, the robot would always drift away gaining speed and tipping over. Playing with the dreaded thing which would not stand at all, and wiggle as some drunk I recalled, that on many YT videos (
[user upgrdman among my favorites on the subject](https://www.youtube.com/watch?v=-bQdrvSLqpg)) people was driving their robots on soft surfaces like carpets and that made me wonder. So I put my robot on a blanket and it helped a little. It reduced both oscillations and the drifting. I have a theory, that firm surface (and/or soft tires) first damps vibrations, but also when a tyre flattens under the weight, it somewhat blocks further movements. Hard tires do not have this effect and on hard surfaces, they spin every time the robot is only slightly out of its balance, where as on soft ones this slight error can be counteracted by a deformed wheel to some very small extent. Think of dry sand on the beach and big soft wheels, I think even when unpowered, the robot could have a chance to stand still in such an environment. So it thought about changing the wheels (and tires) because at that time I used pretty hard wheels from a hardware store (some furniture ones) and I ordered 70mm and 110mm squishy wheels for model aircraft. And then, while waiting for the wheels I made hasty decision to shorten the thing.

### Length
Bear in mind, that I can be completely wrong here! As far as I understand, when you have a normal pendulum, the longer the string is, the faster will be the linear velocity of the pendulum bob. I assumed The same thing will be with inverted pendulum, thus the taller the robot, the bigger velocity of it top-end when tipping over. Thus, I concluded, the faster lower-end movements will be necessary to counteract the moving top. At that time I thought my motors are to slow, so It seemed to be a good idea to shorten the body. So I did that. And the improvement was negligible if at all. Also, from the pendulum frequency equation I knew, that shortening the length increases the pendulum frequency, so I understood that my algorithm must react faster from now on.

### Mechanical issues
Then I realized, that the wheels got loose. I ordered a bunch of adapters and fixed them in place. Still no luck.

### Scratching head and tuning PID
I couldn't sleep but visualized my dangling robot. I tried to tune the PID controller a lot. And still, every time I increased Ki it oscillated, and when decreased it drifted away. Too much Ki and it oscillated left and right, and too much Kd and it oscillated but in like one direction. Like hiccup. Derivative term damps output when error decreases, so in theory, the robot should decelerate near sweet point and make a full stop but instead, it would stop for a moment (like millisecond moment), and start in the same direction, then stop and then again, all in sudden sequence.

### nRF24L01+ distraction
I was frustrated and sick and tired, so decided to do something different, as a break from the main subject. I got into nRF24 code and telemetry. I wanted to make full telemetry for the project as YT user `upgrdman` did. I thought that it would help me to debug. Then I run into more problems, procrastinated a little bit more with refactoring my SPI library and so on. I had curious adventures with nRF though, but this is completely different story. I use an excellent application 
[called KST2](https://kst-plot.kde.org/) for plotting the data. From the plots, I observed that the robot is drifting due to sudden integral peaks. It's like the robot was balancing for a moment, and then the pitch plot would show a slight shift towards one direction, and just after that the integral part grew, robot started to move and tried to catch up, the *i* increased and increased, robot speed up, and then output saturated leading to a crash. The only thing which helped a little was to increase *Ki* (it amplified the reaction for integral, so robot drove faster, so was able to catch up and straighten), but it also increased oscillations.

### Main loop frequency
In the act of desperation, not hoping for a change for better, I increased the main loop frequency from 100Hz to 1kHz. I could do that because I use very fast STM32F4 (compared to Arduinos and Atmegas that everybody seems to love so much) with a fpu. And that was it. It calmed the robot so much, that I could increase Ki to the values not possible before. Then I mounted the 110mm squishy wheels and it helped even more for the reasons described above, and because bigger wheels give more speed. That's all for now, I'm a little bit tired of this project, but in the future I plan to:

* Program encoders.	
* Program remote control (I bought myself a nice Devo 7E transmitter).	
* Experiment with all (or most) of the parameters I talked about. Height vs. loop frequency, various wheels and so on.
  
![Wheels galore](/assets/IMG_20180111_215004-1024x342.jpg)
