---
title: "Testing Security Detections with Cribl"
date: 2021-07-08T16:25:38-04:00
---

## Background

In my career I have worked with a lot of developers that preach the gospel of Test Driven Development. For those who may not
be familiar, TDD is the idea of creating tests for your code's use cases, often even creating the test before you write the code. 

But what about security detections? We know what an attacks look like, and we can certainly identify something like a password spray attack in our logs, but until we either simulate the attack ourselves or are *actually* attacked we don't truly know if our detections work as expected. (And spoilers for my story: Sometimes they very much don't)

## Enter Cribl

[Cribl Logstream](https://cribl.io) is a product introduced to me by a colleague as a way for us to reduce and improve our logs before they flow into our SIEM. Cribl can do things like sample events that repeat often in logs to reduce unwanted duplicate events, format data into more friendly formats such as JSON, and route one source to multiple destinations among many many other useful things. (Just to be clear, Cribl is not sponsoring this post. It's just a great product worth writing about)

The feature that stood out to me was the concept of a "datagen". A datagen allows the user to capture log events and replay them while doing things like testing new routes or pipelines, adding new fields, or any number of other tasks. If we route those replayed events to our SIEM then we have a way to test our detections without having to simulate attacks against ourself every time we change a detection, or worse, wait for an attack to happen.

## Creating a Datagen

In order to create a datagen we are going to need some events. When I decided to prove out this use case I had password sprays on the brain so I decided to password spray our test Okta tenant. I found a handy, but not recently updated, Python script that was designed to perform password sprays against Okta. You can find the version I forked and made some updates to on my Github at [https://github.com/bseb/Okta-Password-Sprayer](https://github.com/bseb/Okta-Password-Sprayer) . I generated some fake usernames, and fired it off at our test Okta tenant to generate my events. Once completed I logged into our Splunk SIEM and searched out the events from the script and exported them as "raw events". I am not sharing the search or any screenshots from this part of the process as to not divulge too much about my current employer's internal workings, and this should work with any SIEM. **I do highly recommend obfuscating or renaming any fields in the events that may be considered confidential.** Only include what is needed to make your alert fire, and replace the rest with dummy data.

Alterntatively, I have uploaded sample data and Cribl configuration in my [Github](https://github.com/bseb/CriblPasswordSprayPipeline) 

Once you have your exported events in hand head over to [Cribl Cloud](https://cribl.cloud/) and create an account if you have not already. Once Logged in You'll want to click on "Pipelines", then "Sample Data" and "Datagens" in the right side pane.

![CreateDataGen](CreateDataGen.png)

From here click "Attach" and select your exported events file to upload. If you are following along with the examples here name the file passwordspray.txt. I set the "events per minute" field to 20.

## Routing Events from the Datagen

You will also find in the [Github Repo](https://github.com/bseb/CriblPasswordSprayPipeline) a Cribl pack that adds some fields that are required for the simulated data to work with the built in Splunk Okta TA, namely the appropriate source and sourcetype. Adding this pack is pretty simple, click on "packs" at the top of the page and then "add new, import from file" in the upper right.

![UploadPack](AddPack.png)

I wont duplicate Cribl's documentation here, but after configuring the pack and the datagen, the final step is to route the output to your destination of choice. My examples are tailored to Splunk, but could easily be configured for other SIEMs. You'll want to keep this data source disabled unless you are actively testing detections, otherwise you'll find yourself constantly getting password spray alerts, as well as logging a lot of fake events.

## Lessons Learned

When I started this I was looking for a way to simulate events, and prove out some testing framework for detections we write. Unexpectedly I also turned up what could be called a bug in the Okta password spray detection we were using. Out of the box the detection only found failed logins with known good userids. Since the userids I was testing with were all made up the detection completely missed them. The problem is the detection was looking for `INVALID_CREDENTIALS` in the Okta logs but that message only appears if the user is valid. `VERIFICATION_FAILED` was logged when the user was not found. If an attacker was performing a blind password spray attack, we might miss an opportunity to block it before they came back with a better list of users.

![DetectBefore](DetectBefore.png)

![DetectAfter](DetectAfter.png)

This find illustrates the need for a more structured way to test our detections. It is not enough to pull together theoretical indicators of compromise or attack and put them into a detection. Real world simulation is vital to keeping one step ahead of the bad guys.
