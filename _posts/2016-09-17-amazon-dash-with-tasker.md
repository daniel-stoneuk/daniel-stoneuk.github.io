---
published: true
layout: post
date: '2016-09-27 23:14 +0100'
title: Amazon Dash with Tasker
---
## Amazon-Hyphen. 

So, you've probably seen the [Amazon Dash Button][dash] - a compact little wifi connected button. There have been loads of different "hacks" circulating on the internet, and since they were recently released in the UK I decided to order one.

One day later I had it toggling my Lifx light bulb thanks to the countless tutorials on the internet. I followed [Steven Tso's blog post][tso] and it only took me about half an hour.

Setting up the Dash Button with the app was simple, and cancelling at the final step (selecting the product) let's you press it without ordering anything. Since the Dash only connects to WiFi when it's pressed, the majority of the "hacks" online involved monitoring the WiFi network, waiting for the Dash Button to join, and then executing the relevant code.

Works great... Until your internet dies. I don't know about you but I have to restart my WiFi router almost daily to avoid slowing down, which seems to break the ARP monitoring code. When trying it again today, I realised that I could utilise the pointless notification that appeared on my phone instead of ignorantly swiping it away. It works quicker, and works whenever my phone is on and connected to the internet. Essentially 24/7.

![amazondash.gif]({{site.baseurl}}/assets/amazondash.gif)

## How?

As tasker has a plethora of plugins, I reinstalled it for the first time after purchasing it two years ago. Great - it hasn't changed a bit. From here what I did was:

1. Listen for a notification from Amazon Shopping
2. Remove this notification
3. Run plugin to toggle lights (or do anything with tasker)
4. Notify that the lights were toggled.

Nice and simple. I'll walk you through each step with the screenshots below.

### Prerequisites

I expect you to have the Amazon Shopping app installed and to have setup the Dash Button (skipping the last step), with Dash Button Notifications switched on. 

### Step 1 - Listen

Let's start off by downloading a few apps: [Tasker (£2.99)][tasker], [Notification Listener (free with IAP)][notilisten] and [Auto Hue (79p - my plugin of choice)][autohue]. 

Open tasker, accept the disclaimer and then press the big `+` button at the bottom. Tap on Event > Plugin > Notification Listener > Notification Listener. Enter the plugin configuration by tapping the Edit icon. Select notification event "Posted" and select the Amazon Shopping app.

![tasker-1.png]({{site.baseurl}}/assets/tasker-1.png){:.screenshot}

Tap the tick and move on. Press back to save the configuration and tap New Task & give it a name. 

### Step 2 - Cancel

Now we have intercepted the notification, we can clear it. Tap the `+` button and navigate back to the Notification Listner plugin. Tap on `Cancel notificaions` this time. Enter the configuration screen and choose "By title" in the dropdown. Enter `%nltitle` in the text field.

![tasker-2.png]({{site.baseurl}}/assets/tasker-2.png){:.screenshot}

Save the configuration and proceed with step 3.

### Step 3 - Act

Awesome. We can now do whatever we like with the tasker task. I have my task setup so that it will:

1. Create a notification to indicate that the Dash Button press was received.
2. If my lights are on, then turn them off and replace the notification saying the lights were toggled.

![tasker-3.png]({{site.baseurl}}/assets/tasker-3.png){:.screenshot}

The notification looks like this, in case you were wondering: 

![tasker-4.png]({{site.baseurl}}/assets/tasker-4.png){:.screenshot}

Anyway, that rounds off this pretty long guide. I applaud you if you got this far, and I hope I've helped ya.

Ttyl,  
Dan

[dash]:	https://www.amazon.com/Dash-Buttons/b?ie=UTF8&node=10667898011
[tso]: http://steventso.com/amazon-dash-lifx/
[notilisten]: https://play.google.com/store/apps/details?id=com.balda.notificationlistener
[tasker]: https://play.google.com/store/apps/details?id=net.dinglisch.android.taskerm
[autohue]: https://play.google.com/store/apps/details?id=com.cuberob.autohue
