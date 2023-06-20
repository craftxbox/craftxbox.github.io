---
layout: post
title: Secure your Yubikey Static Passwords
date: 2023-06-19 17:19:00
author: craftxbox
keywords: yubikey, yubico, 2fa, mfa, multifactor, xor, encryption, nfc, rfid, open-source
---

Recently, I picked up two Yubikey 5 NFC security keys from a promotional offer with Cloudflare.  
These things are great!  Sporting not just the standard FIDO2/U2F capabilities you come to expect from a security key, they come with other great features; such as  
1. Storage for 32 traditional 2FA TOTP secrets
2. A OpenPGP Smartcard interface (great for git commit signing!)
3. A PIV Smartcard interface

>##### > PIV doesn't see much usage, unless you want to use your Yubikey as a portable CA. (me)<br/>&nbsp;&nbsp;&nbsp;Or you're the type of masochist to migrate their home workstation to AAD for smartcard logins. (also me)

Aswell as the subject of this post, Two configurable HID-Keyboard slots.

![Yubikey manager showing the two OTP slots](https://transfur.science/7t6g74yp)

Now by default, Slot 1 is programmed with Yubico's own "Yubico OTP" protocol, which generates 44 character long One Time Passwords, for use with validation with their own servers.  
This is neat, but it's not very useful to me, as it's not compatible with many services, and it's proprietary to only Yubico devices.  
So instead, I decided to repurpose the slot to store a static password, which I could use anywhere I wanted.

![Yubikey manager showing the programming of a static password](https://transfur.science/0ufbk2f)

This was perfect, I now had a long, hard to guess and remember password that was always with me.  
Of course, It was vulnerable if someone got a hold of my Yubikey, but I figure: If someone has stolen my Yubikey, I have much bigger problems to worry about.  
Otherwise, it was secure. Right?

Well, not quite.

## Enter: The NFC interface.
![Yubikey manager showing the Interfaces available on a Yubikey. The NFC OTP interface is highlighted.](https://transfur.science/sllou7gu)  

The latest Yubikey models have an NFC interface, which allows you to access one of the two OTP slots from your phone, without having to plug in the Yubikey.  
This is *intended* for use with their Yubico OTP protocol[^0], but it works with any of the OTP settings...  Including the static password I programmed into slot 1.  
![NFC Tools showing the output of a Yubikey programmed with a static password over NFC.](https://transfur.science/m1rty6y7)  

This doesnt look like any kind of data, but it is: It's the static password, encoded in Keyboard Scancodes.  
![NFC Tools showing the raw hexadecimal payload of the NFC data.](https://transfur.science/h8nu7c8g)  
But what good is this? It's just all garbage! We can't use this as a password! Well, Not without some annoying workarounds.  
But there's a bigger issue: This is NFC. **Anybody with a NFC enabled phone can read the data, Use those workarounds themselves, and get your password!**  
Or conveniently, they don't even need to use the workarounds, as Yubico has [kindly provided an android app to do just that!](https://play.google.com/store/apps/details?id=com.yubico.yubiclip)

So much for security... But this isn't really Yubico's fault. This feature was designed for their own OTP protocol, and it's just a side effect that it works with *any* OTP slot type.  
Infact, after thinking for a while, It's quite convenient, because we can use it to our advantage!

## The Solution

XOR is a simple encryption algorithm, which takes two inputs, and produces an output which is the result of the two inputs being Bitwise-XOR'd together.
It's also a reversible algorithm, meaning that if you XOR one of the inputs to the output, you get the other input back!    
Aswell, it's in theory, [**completely secure**](https://en.wikipedia.org/wiki/XOR_cipher#Use_and_security:~:text=With%20a%20key%20that%20is%20truly%20random%2C%20the%20result%20is%20a%20one%2Dtime%20pad%2C%20which%20is%20unbreakable%20in%20theory.).

So, why don't we use this to encrypt our static password?  
First problem: We need to get the password into the Yubikey. The Yubikey manager doesnt support *binary* data, as an XOR operation would give us, Only letters on a keyboard.[^1]  
This isnt too much of a problem, We can encode the password in Base64, and then use the Yubikey manager to program it in. (though, we lose some password bits in the process)

Second problem: We need to get the password out of the Yubikey. The NFC interface only outputs the password as a series of keyboard scancodes, which we can't use directly.  
But! We can just use that Yubiclip app from before instead, then XOR the output with the password we programmed in, and we get the password back!  
And as any good piece of software is, [Yubiclip is open source!](https://github.com/Yubico/yubiclip-android)

And so, I decided to [fork it.](https://github.com/craftxbox/yubiclip-xor)
The first step was to add the basic POC functionality. This took me *hours* of pain, all because I didn't blindly trust the code of an [answerer on StackOverflow.](https://stackoverflow.com/a/8235530/3934270)[^2]
```java
    xorKey[i] = (byte) Integer.parseInt(xorKeyString.substring(stringIndex, stringIndex + 1), 16);
```
Simple enough line of code, Take a hex character, and convert it to a byte. Can you see the problem?
```diff
- xorKey[i] = (byte) Integer.parseInt(xorKeyString.substring(stringIndex, stringIndex + 1), 16);
+ xorKey[i] = (byte) Integer.parseInt(xorKeyString.substring(stringIndex, stringIndex + 2), 16);
```
One number. One character. One byte. String::substring() is *inclusive* of the first index, and *exclusive* of the second index.  
My code was only taking the first character of the hex string, and ignoring the second. Resulting in half the key being ignored.  
As the saying goes, "If it ain't broke, don't fix it."

>##### > Though, it's still good not to trust everything you copy from StackOverflow!

After that nonsense, I was able to get the basic functionality working. But I wanted to make it more user friendly.  
I had been using the Yubikey Personalization Tool to load the password into the Yubikey, but it was a bit of a pain to use.
Aswell, I had to do the initial XOR step manually, aswell as the Base85[^3] encoding step, Then to hex bytes to throw in the 'scancodes' box in the tool.
>##### > On second thought, the whole 'binary to text' step was probably unnecessary, It was only the Yubikey manager that was complaining about the binary data, and I ended up writing raw bytes to the Yubikey anyway. <br/>&nbsp;&nbsp;&nbsp;But, at the time I didn't know that the strings could handle binary data, so I just went with it.

So I made a python script for it. Yubico has a [python library](https://developers.yubico.com/yubikey-manager/Library_Usage.html) for management of the Yubikey, so I used that to Apply the XOR, Encode the result, and write the password to the Yubikey.  
But then I thought, Generating and loading the XOR key into the app wasn't particularly trivial either. To do it securely required a *nix system[^4] and `hexdump`ing /dev/urandom
```bash
$ hexdump -vn30 -e'30/1 "%02X" "\n"' /dev/urandom
```
So I made the script do that too. Then I realized getting that *onto* the phone was a pain too... But we have an NFC tag we can load arbitrary data onto right here!  
From there, my simple script evolved into a full fledged tool, which can generate a secure key, shuttle it to the phone via the NFC Interface, and then load the final password into the Yubikey.  
No thought required. [Just run the script, and it does everything for you.](https://github.com/craftxbox/yubiclip-xor/blob/master/py/xorsetup.py)  

## What can I do?

If you've **never set a static password** on your Yubikey, you can obviously just **ignore this**. By default the only slot programmed contains the factory key for Yubico OTP.  
While that *could* be a problem if your threat model includes people scanning your key to steal your Yubico OTP tokens, not many services use it for 2 factor to begin with, and you can always disable the NFC interface entirely if you're worried about it.  

If you DO use a static password, but you put it on the **Long-press slot #2**, you should also be okay, provided you have not *manually* changed the slot to be presented over NFC.  
By default, the Yubikey will only present the Short-press slot #1 over NFC. So you should be okay, barring the Yubico OTP note above.  

Finally, If you **BOTH** have a static password, *and* it's on the **Short-press slot #1** (or manually changed NFC to #2) you are **at risk** of this issue.  
If NFC isn't useful to you, My recommendation is to just disable it entirely through the Yubikey Manager.  
However, if you need your password on-the-go and don't want to carry a USB OTG adapter everywhere, I have released my work above as an open source project.

You can use [this Python script](https://github.com/craftxbox/yubiclip-xor/blob/master/py/xorsetup.py) to set-up the key.
Then, you can install [the Android app](https://github.com/craftxbox/yubiclip-xor)[^5] to get the password back out of the Yubikey.

Eventually, the goal is to merge this feature into the newer, cross-platform [Yubico Authenticator](https://github.com/Yubico/yubioath-flutter) app, but I haven't had the time nor motivation to do so.

***
### Footnotes
[^0]: All the other apps on the card can be accessed over NFC too, for what it's worth. But OTP is the only one that presents itself as a tag.
[^1]: This is because the Yubikey is programmed to send the data as a HID Keyboard, which is limited to the characters available on, well, a keyboard. There are some particular snafus with keyboard layouts, but in general, it's limited to the characters on a keyboard.
[^2]: Yes I know this question was for C# and not Java, but the logic flow is the same.
[^3]: Yes, Base85. Specifically Ascii85 (and not the ZeroMQ variant). It let me use a full 30 characters of password with the limited, 38 byte payload of the Yubikey. I could have used Base64 and gotten 28 password characters, but 30 was too nice of a number to pass up.
[^4]: I found out later that Powershell could also do the job, but the command wasn't very friendly looking like the *nix one is... Oh well, I like bash better anyways.
[^5]: Sorry iOS folk. I don't have XCode or a developer's license to make an app for you.