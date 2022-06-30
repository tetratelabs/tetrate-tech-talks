---
# About Tetrate Tech Talks

A series of broadcasts that runs every Friday at 9:00 AM Pacific
  streamed to YouTube and to LinkedIn Live.

##https://tetratelabs.github.io/tetrate-tech-talks/

Objectives:

- Learning and sharing knowledge
- Getting to know the diverse members of the service mesh community
- Have fun!

Episodes present a technical topic and feature a guest.

---
# I'm your host, Eitan Suez

I work at Tetrate, and I like to talk tech.

If you're with us live, I invite you to say hello to everyone in the chat.

---
# Previously on Tetrate Tech Talks..

- June 17 - [Let's talk service mesh with Kelsey Hightower](../../episode11/)
- June 24 - [A conversation with Josh Long](../../episode12/)
- All past episodes are on our [YouTube playlist](https://www.youtube.com/playlist?list=PLm51GPKRAmTlOkjWDJBQYtjcc9WPk4E4F).

---
# Our episode..

## Learning Istio from the Inside

Way back in [Istio Weekly episode 12](https://youtu.be/o3Fi6nwuuiI), our guest [Aditya](https://www.linkedin.com/in/aditya-prerepa-963007178/) suggested learning Istio less from the documentation and more by studying the source code and its automated tests.

In this episode, we explore that avenue.  We demonstrate setting up a workstation for local development with Istio.  We begin by checking out the code, building it, building the docker images and pushing them to a registry, and finally installing Istio on a local k8s cluster using the published images.  Next we demonstrate ways of speeding up the development cycle by running istiod directly from our workstation, bypassing the docker image building and publishing steps.  We then show how to configure the Istio codebase in an IDE and step through the code in debug mode.  We finish with a discussion of the basics of reading and executing a simple unit test.

This episode is the first in a contemplated series where subsequent episodes will walk through sections of the codebase, leveraging the setup demonstrated here.

---
# References

- [Istio wiki - Using the codebase](https://github.com/istio/istio/wiki/Using-the-Code-Base)
- Local Istio Development [IstioCon 2021 session](https://youtu.be/g4A8LAauyJA) and accompanying [repository](https://github.com/howardjohn/local-istio-development)
- [Goland IDE](https://www.jetbrains.com/go/)
- [Testing golang with httptest](https://speedscale.com/testing-golang-with-httptest/)
- [Gomega BDD test framework](https://onsi.github.io/gomega/)

---
# Upcoming Episodes

- Friday July 8: Episode 14: Tetrand Profile: Liyi Huang, from Tetrate's Customer Success team

---
# Istio Certification

- [Certified Istio Administrator by Tetrate](https://academy.tetrate.io/courses/certified-istio-administrator)

    If you already obtained your Kubernetes certifications, the next logical step is Istio certification.
    When you're ready, we have the certification exam available for you to take.

---
# Join the conversation..

Here is the [Invite URL](https://tetr8.io/tetrate-community) to our community Slack organization https://tetrate-community.slack.com/

