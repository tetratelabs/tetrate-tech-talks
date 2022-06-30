# Episode 13

[LinkedIn](https://www.linkedin.com/feed/update/urn:li:ugcPost:6945884915657830400/) | [YouTube](https://youtu.be/ueTXrMmWklY)

## When

Friday 2022.07.01 @ 9:00 AM Pacific time / 11:00 AM Central time

## Your Host: [Eitan Suez](https://www.linkedin.com/in/eitan-suez-2336b26/)

Eitan will be hosting this episode.

## Topic:  Learning Istio from the inside

Way back in [Istio Weekly episode 12](https://youtu.be/o3Fi6nwuuiI), our guest [Aditya](https://www.linkedin.com/in/aditya-prerepa-963007178/) suggested learning Istio less from the documentation and more by studying the source code and its automated tests.

In this episode, we explore that avenue.  We demonstrate setting up a workstation for local development with Istio.  We begin by checking out the code, building it, building the docker images and pushing them to a registry, and finally installing Istio on a local k8s cluster using the published images.  Next we demonstrate ways of speeding up the development cycle by running istiod directly from our workstation, bypassing the docker image building and publishing steps.  We then show how to configure the Istio codebase in an IDE and step through the code in debug mode.  We finish with a discussion of the basics of reading and executing a simple unit test.

This episode is the first in a contemplated series where subsequent episodes will walk through sections of the codebase, leveraging the setup demonstrated here.

See you there!