---
layout: project
title: DM Soundex
description: token filter for lucene
img: /img/logo-dmsoundex.png
link: https://github.com/netomi/dm-soundex
role: owner
license: Apache license 2.0
category: misc
tags: java lucene codec
---

This project was a contracting work to add an implementation of the <a href="https://en.wikipedia.org/wiki/Daitch%E2%80%93Mokotoff_Soundex">Daitch-Mokotoff Soundex codec</a>
to the <a href="https://commons.apache.org/proper/commons-codec">Apache Commons Codec</a> project in order to use it as a token filter within lucene.

The project available at github is a standalone version to be used together with lucene.

The actual implementation was merged into the <a href="https://github.com/apache/commons-codec/blob/master/src/main/java/org/apache/commons/codec/language/DaitchMokotoffSoundex.java">commons-codec repo</a>.