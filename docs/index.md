# Introduction

In the latest releases of Istio, with support for Ambient mode becoming Generally Available (GA), and with its support for the Kubernetes Gateway API, we are finally at a point where platform engineers can provision, configure and control network traffic in uniform way, and without the overhead of sidecars.

It is important to stress that the above term "network traffic" implies traffic in all directions: at ingress, at egress, as well as internal "mesh" traffic, also known as "east-west" traffic.
This vision of a single, uniform API to control traffic in all directions is something we at Solo.io have dubbed "Omni" (see [What is Omni?](https://www.solo.io/resources/video/what-is-omni-part-1-of-2)).

This document is a hands-on exploration of configuring ingress, egress, and mesh traffic, in order to see first-hand how this is done.
The main objective is to get a sense for how these different activities are performed with a single API, and where the process of configuring and controlling traffic is in many ways the same.
