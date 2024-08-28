---
sort: 2
---
## Introduction

Production or test time cost reduction is important. 

The following document will describe how to minimize flashing time by combining binaries into one  to single the flash action.

This may be usefult for example to combine:

- bootloader + application + production reset application image
- bootloader. + production tests application + actual application

This can be extended to many combination but the goal is mainly to explain the procedure to combine a GBL (OTA image) with the other binaries as this one does not contains its landing address.
