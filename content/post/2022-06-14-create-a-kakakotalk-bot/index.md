---
title: How to create a KakaoTalk Chat Bot
description: There is almost zero content in english on how to use the KakaoTalk APIs. Good for you I got it figured out and we will explore it here and create a see how we can create a KakoTalk Chat Bot that translates everything we write into Korean or English.
slug: how-to-create-a-kakaotalk-chat-bot
date: 2022-06-14 00:00:00+0000
image: cover.png
categories:
- dev
tags:
- dev
- ruby
---

Interested in creating a KakaoTalk Chat Bot or explore their APIs but unfortunately you can't read nor speak Korean? Well, this is the place for you.

I recently discovered KakaoTalk via a couple of South Korean friends that use it for their daily communication. Unfortunately, one of them recently moved to the US and still has a bit of difficulty with communicating in English.
As a non-native english speaker, I remember how long it took me to learn it and I wished I just had a quick way to still chat with my friends without having to go on Google Translate every 5 seconds to understand what they are writing.
My friends also often write a bunch of messages in group chat in Korean and as someone who understands absolutely nothing, it started to be a bit of a hassle to translate everything into English myself.

Here's where that idea of creating a KakaoTalk Translation Bot came from.

Bad news, most of the KakaoTalk users are Korean and therefore, when I decided to create that chat bot, there was no resources available in english concerning any kind of KakaoTalk API.

You can see what it looks like in action here:
![Example](/images/Screen%20Shot%202022-03-08%20at%209.36.04%20PM.png)

As you can see the bot can translate from English to Korean and from Korean to English.
It also directly attaches the translation as a reply to the message.

So how do create that?

## KakaoTalk Unofficial API

KakaoTalk possess a various list of APIs available here: https://developers.kakao.com/.
The 2 closest APIs we could use to create a Chat Bot could be this one https://developers.kakao.com/product/kakaoTalkChannel or this one https://developers.kakao.com/product/message.
Unfortunately, nothing, at the time this blog post was wrote, exists to directly reads a user conversation and reply to it.

KakaoTalk uses the proprietary LOCO message protocol for its messaging system. I was not successful in finding any good english resources about it but a few unofficial LOCO-compatible projects exist on Github.

The one I uses for my translate bot is https://github.com/storycraft/node-kakao and the thing that caught my attention is this example: https://github.com/storycraft/node-kakao/blob/b3665855481e631f29fad813cd954b29279568e3/examples/login-chat.ts#L30.
Looks like that little piece of code is exactly what we want to do: append a text message to a conversation and send it!

## Let's code our Chat Bot

Now that we found our potential way to interact with KakaoTalk it's time to develop our translation bot.
I personally use [Papago](https://papago.naver.com/) and its [API](https://developers.naver.com/docs/papago/README.md) as my translation engine. You can also use Google Translate or Amazon Translate. The translations won't be good but I find Papago to be the one that sucks the least.

You will also need to create a new KakaoTalk account and use it as your bot account.

### UUID Generator

You also need to get or generate a device UUID. 
Create a new node project (`npm init`), add node-kakao (`npm i -S node-kakao`) and create a file with this script
```typescript
# uuid_generator.ts

import { util } from 'node-kakao';

console.log(util.randomAndroidSubDeviceUUID());
```

Run `npx ts-node uuid_generator.ts` in your terminal and get your UUID.

### API Connection

Now let's try to connect to the API. In a new file you can add this code:

```typescript
# server.ts

import { AuthApiClient, TalkClient } from 'node-kakao';

const DEVICE_UUID = ""; // Put the UUID you generated
const DEVICE_NAME = ""; // Any name works here

const EMAIL = ""; // Put your KakaoTalk account email
const PASSWORD = ""; // Put your KakaoTalk account password

const CLIENT = new TalkClient();

async function main() {
    const api = await AuthApiClient.create(DEVICE_NAME, DEVICE_UUID);
    const loginRes = await api.login({
        email: EMAIL,
        password: PASSWORD,
    });
    if (!loginRes.success) throw new Error(`Web login failed with status: ${loginRes.status}`);

    console.log(`Received access token: ${loginRes.result.accessToken}`);

    const res = await CLIENT.login(loginRes.result);
    if (!res.success) throw new Error(`Login failed with status: ${res.status}`);

    console.log('Login success');
}
main().then();

```

Run the script with `npx ts-node server.ts`. If everything went well you should see `Login success` in your terminal.
Good! Now let's see if we can collect the messages sent to our bot.

### Catch messages

Here we will use the function `CLIENT.on('chat', ...){}` which means ANY message sent to the bot will be processed.
For each message received, we will send back a direct reply with `OK`.

```typescript
# server.ts

import { AuthApiClient, TalkClient, ReplyContent, ChatBuilder, KnownChatType } from 'node-kakao';

const DEVICE_UUID = ""; // Put the UUID you generated
const DEVICE_NAME = ""; // Any name works here

const EMAIL = ""; // Put your KakaoTalk account email
const PASSWORD = ""; // Put your KakaoTalk account password

const CLIENT = new TalkClient();

CLIENT.on('chat', async (data, channel) => {
    const sender = data.getSenderInfo(channel);

    console.log(data)

    if (!sender) return;

    if ('text' in data && Object.keys(data.attachment()).length === 0) {
        channel.sendChat(
            new ChatBuilder()
                .append(new ReplyContent(data.chat))
                // @ts-ignore
                .text("OK")
                .build(KnownChatType.REPLY)
        )
    }
});

async function main() {
    const api = await AuthApiClient.create(DEVICE_NAME, DEVICE_UUID);
    const loginRes = await api.login({
        email: EMAIL,
        password: PASSWORD,
    });
    if (!loginRes.success) throw new Error(`Web login failed with status: ${loginRes.status}`);

    console.log(`Received access token: ${loginRes.result.accessToken}`);

    const res = await CLIENT.login(loginRes.result);
    if (!res.success) throw new Error(`Login failed with status: ${res.status}`);

    console.log('Login success');
}
main().then();
```

Run the script with the same command as above. Add your bot as a friend to KakaoTalk and send him a message. He should reply to you with OK.
Alright now the final part, let's use the Papago API to translate the messages.

### Add the Translator

Add the papago and google translate libraries: `npm i -S papago @google-cloud/translate` and go create a Naver developer account here: https://developers.naver.com/main/. You will need to generate a Papago API key.
I'm using Google Translate here to detect the source language.

Now let's add a translation function to our script and use it in our catch message function:

```typescript
# server.ts

import { AuthApiClient, TalkClient, ReplyContent, ChatBuilder, KnownChatType } from 'node-kakao';
import {Translate} from "@google-cloud/translate/build/src/v2";
import Translator from 'papago';

const DEVICE_UUID = ""; // Put the UUID you generated
const DEVICE_NAME = ""; // Any name works here

const EMAIL = ""; // Put your KakaoTalk account email
const PASSWORD = ""; // Put your KakaoTalk account password

const CLIENT = new TalkClient();
const translate = new Translate();

async function translateText(text: string) {
    let languageDetection = await translate.detect(text)
    let output = "ko"

    console.log(languageDetection)

    if (languageDetection[0].language === 'ko') output = "en"

    let translator = new Translator("PAPAGO ID", "PAPAGO API KEY")
    return translator.translate(text, languageDetection[0].language, output)
        // @ts-ignore
        .then(res => {
            return res.text
        })
}

CLIENT.on('chat', async (data, channel) => {
    const sender = data.getSenderInfo(channel);

    console.log(data)

    if (!sender) return;

    if ('text' in data && Object.keys(data.attachment()).length === 0) {
        let translation = await translateText(data.text)
        
        channel.sendChat(
            new ChatBuilder()
                .append(new ReplyContent(data.chat))
                // @ts-ignore
                .text(translation)
                .build(KnownChatType.REPLY)
        )
    }
});

async function main() {
    const api = await AuthApiClient.create(DEVICE_NAME, DEVICE_UUID);
    const loginRes = await api.login({
        email: EMAIL,
        password: PASSWORD,
    });
    if (!loginRes.success) throw new Error(`Web login failed with status: ${loginRes.status}`);

    console.log(`Received access token: ${loginRes.result.accessToken}`);

    const res = await CLIENT.login(loginRes.result);
    if (!res.success) throw new Error(`Login failed with status: ${res.status}`);

    console.log('Login success');
}
main().then();
```

Now run the script, send a message and you should your message being translated to Korean!
If you send a message in Korean you should also see it being translated to English!

Enjoy your new KakaoTalk translator with your friends.

## Sources

You can find the whole project on my Github repo here:
https://github.com/tgeselle/kakaotalk-translator