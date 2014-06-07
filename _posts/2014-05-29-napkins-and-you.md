---
layout: post
title: Napkins and you
description: "Ascii diagrams and how not to love your users"
modified: 
tags: [Ascii, perl, SaaS]
image:
  feature:
  credit:
  creditlink:
comments: true
share: true
---

Heres a short post for today. A while back I remember being very excited to see that asciiflow.com had updated their platform and the tool had become far richer.

What I did not realise at the time was that the source code which once use to be available had vanished from the internet (aka github). I am not exactly sure what the motivation for this would be since its either a thinly veiled attempt to siphon up technical details from those less security concious or just a plain change of heart. Either way it left me feeling fairly disappointed in the project.

## Learning to love perl - asciio

After being faced with the need to document things quickly and in an ever changing way I went on the hunt for a replacement to asciiflow and I found [**asciio**](http://search.cpan.org/dist/App-Asciio/lib/App/Asciio.pm) lurking in the Fedora repositories.

I've been using asciio now for a few weeks and can say that this application is simply awesome! A GTK interface to interact with layer based ascii diagrams is exactly what I was after. Most particularly I can be confident that these documents stay with me and my /dev/sda.

## Some assembly required

There is mostly undocumented feature in the save and save as function. That being the all documents saved with **.txt** extension are NOT able to be loaded as these are the ascii exports.

To save a document to reuse give it another extension. Personally I have used **.asciio** to keep things obvious and not had any future trouble. Details are in [BZ1097538](https://bugzilla.redhat.com/show_bug.cgi?id=1097538).

Happy documenting!
