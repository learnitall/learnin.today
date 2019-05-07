---
title: "Cyber-Clippy: The Need for a Clippy-Like Assistant to Secure Desktops at Home"
categories:
  - Cyber-Clippy
---

If you haven't heard of Microsoft's Office Assistant
[Clippy](https://en.wikipedia.org/wiki/Office_Assistant), then please do yourself
a favor and check it out. Clippy was a little assistant that came bundled with
Microsoft Office for helping users do things like write letters and [resumes](https://www.youtube.com/watch?v=DF-dRqadOZI).
Having now gone down in infamy, Clippy lives on in the realm of memes and at the
butt end of sentences like: *Hey do you remember that thing, from like the '90s?*

Putting everything that makes Clippy Clippy aside for a moment though, the idea
of having a tool for assistance in performing various tasks sounds pretty
powerful- one where users can not only feel empowered to do things outside of their
original comfort zone, but have something there on demand to teach them interactively.
It's like trying to learn how to make pancakes with a chef right there by your
side, creating a sense of comfort.

I think that this idea of an assistant should therefore be executed within the
context of cybersecurity. We have millions of people going out onto the Internet
everyday exposing themselves to various risks and consequences without realizing
them nor preparing for them. A little assistant that can warn users of holes in their
system and then teach them how to fix those holes right there on the spot would be
incredible. **We can empower users who are less technically inclined to become both
cyber-aware and cyber-trained right at home.**

## Cybersecurity Awareness and Training at Work

To put this idea into context, we can take a look at how cybersecurity awareness
and training within an organization is defined and executed. This will help to make
the need for an assistant at home even greater.

[National Institute of Standards and Technology (NIST) Special Publiation 800-50](https://csrc.nist.gov/publications/detail/sp/800-50/final)
"Building an Information Technology Security Awareness and Training Program" outlines
a pipeline for employees that looks something along the lines of this:

![Awareness and Training Pipeline from NIST 800-50](../../assets/images/2018-05-07-cyber-clippy/ath.png)

There are more tiers to this mentioned, specifically I have omitted the "Education"
tier, yet this is the main jist that I feel is important to understand to make
the idea of "Cyber-Clippy" come into view.

Employees move through this pipeline from the bottom to the top.

### Awareness (All-Employees)

[NIST Sepcial Publication 800-16](https://csrc.nist.gov/publications/detail/sp/800-16/final)
"Information Technology Security Training Requirements: a Role- and
Performance-Based Model" defines the awareness tier as a method for focusing
attention on security issues, coming in the form of presentations, posters, banners
on computer screens, etc. The end goal here is to help users **learn of potential
security risks and appropriate behavior that should be implemented**.

Research from Bada et al. *Cyber Security Awareness Campaigns: Why do they fail
to change behaviour?* also mentions that awareness campaigns are more than just
saying what one should and shouldn't do, it involves ensuring that people can **understand
and apply the advice given and are motivated to do so**. They make the point that
motivation for acting securely is a complicated psychological issue, however it must
be addressed. (Be sure to give it a look, it's an awesome read.)

### Security Basics and Literacy (All-Employees)

This tier is specifically outlined in NIST 800-16 as an eight-hour crash course to
give employees the chance to learn the ABCs of security. The goal here is to build
a baseline set of knowledge that encompasses an overview of various training topics,
in order to **create a foundation** for future training. Topics in the ABCs include:

* **L**aws and Regulations
* **R**isk Management
* **T**hreats
* **V**ulnerabilities

It is important to note that NIST 800-16 acknowledges that this tier and the
Awareness tier tend to overlap a bit.

### Role-Based Training (As-Needed, Based on Role)

There is a huge and complicated matrix mentioned for this tier that can be
used for creating a training program or course, however for the sake of this
document what we are concerned about here is the fact that this tier is meant
to **teach people how to perform a certain function for their role**; creating
security skills for a person's role in the organization.

## Cybersecurity at Home

This kind of pipeline just doesn't exist in a home environment. It may be a process
that some go through when working with their computer, yet there isn't a universal,
formal program that all Internet-users are put through.

This seems counterproductive, given that at work, there is an IT security group
dedicated towards securing the end-user desktop systems that everyone uses.
**At home however, a person is 100% responsible for securing their computer yet
not always motivated or skilled enough to do so**. Since awareness and training
programs are built for addressing these pitfalls, it seems logical to try and
find a way to bring them to the home environment.

Additionally, I think it is worth mentioning that at home, awareness and training
mushes to become this gray blob. For example, at work awareness program topics
can include (from NIST 800-50, section 4.1.1):

* Password usage and management
* Protection from viruses, worms, Trojan horses, and other malicious code
* Incident Response
* Social Engineering
* Spam
* Data backup and storage
* Web usage (allowed activity and monitoring)

These topics are all impacted by user behavior, the more social/cultural
aspects to a user's experience at work. Most of the technical details in this
topic however, the details that would require training, are handled by IT or
the security team. Employees don't have to worry about creating a centralized
data backup and storage system, installing anti-virus, setting password policies,
performing incident response, monitoring their on behavior online, etc. There are
people who can do these tasks for them.

This just isn't the case at home. Users at home, who are 100% responsible for all
these topics, can be aware of them and understand their implications, yet cannot
fully change their behavior because they don't have the necessary technical skills
developed. They would have to go through the necessary training to gain the technical
skills needed, and most of the time, the motivation or the time just isn't there
to do so.

## Problem with At-Home Training

This isn't to say that it isn't possible for people at home to teach themselves
security. There are plenty of articles, blog posts and how-tos that go through different
aspects of hardening. If someone was motivated enough, they could for sure start
performing Google searches such as:

* *How do I turn on my firewall?*
* *How do I backup my computer?*
* *How do I perform spam detection?*
* *How do I know if this is safe to download?*
* *How do I use my anti-virus?*

Yet, how does one know when to stop searching? How can one guarantee that all aspects
of computer hardening are covered? With all the different opinions out there on the
Internet, how can one make an informed judgement as to whether a hardening task
(such as [installing a VPN](https://privacy.net/how-to-secure-your-computer/))
is even necessary or worth the cost?

The benchmarks created by the [Center for Internet Security](https://www.cisecurity.org/cis-benchmarks/)
could completely answer all these questions and throw them out the window. These
benchmarks are created by a professional community of cybersecurity experts from
around the world and they contain rich amounts of information that address the goals
of both awareness and training programs. For instance, the Bluetooth recommendations
from the macOS 10.13 benchmark has the following:

* Overview of Bluetooth
* Rationale for implementing recommendation
* Procedure for checking if the recommendation is in place
* Procedure for implementing the recommendation
* Potential negative impacts for using the recommendation

But even these benchmarks aren't fully applicable to this situation, because they
are written "...for system and application administrators, security specialists,
auditors, help desk, and platform deployment personnel..." These benchmarks are rich
with technical mumbo-jumbo and are *huge* in length. Some of the recommendations
may also just be not applicable towards an at-home user, possibly unnecessarily
restricting use of the system.

This is why there needs to be some mentor to facilitate at-home training awareness
and create a middle ground, where users are empowered to learn what they **need**
to learn.

This is where Cyber-Clippy comes in.

## Cyber-Clippy

Cyber-Clippy is an idea to bring together configuration management and technical
benchmarks into an open-source assistant that facilitates cybersecurity awareness
and training for at-home users. It adapts the awareness and training pipeline from
NIST into the following:

![Cyber-Clippy Workflow](../../assets/images/2018-05-07-cyber-clippy/cc.png)

> Same color coding as the NIST hiarchy diagram: Blue - Awareness, Green -
> Security Basics and Literacy, Purple - Training

We start by creating a benchmark for an end-user system and auditing for compliance
against it. With this, we can generate a report for the user and providing a rating
for their system (say on a scale from 0%-100% with colors to emphasis good and bad).
This report will contain holes that the scan found, and for each hole, describe the
following (modeled after the CIS benchmarks):

* Awareness:
  * Technologies involved with the hole (i.e. firewall, antivirus)
  * Why the hole is bad and possible consequences
  * Future behavior that should be implemented
* Training:
  * Steps to take for fixing the hole
  * (If applicable) Description of how to use the technologies that fix the hole
    (i.e. firewall, antivirus).

The language used to describe the above should be the same kind of language used
during the Security Basics and Literacy tier: targeted towards beginners trying to
build a foundation in cybersecurity basics. Word choice should be simple and focused,
while using important terminology found in more technical document(s)ation.

Feel free to send me an email with thoughts, concerns or suggestions to help
develop this idea further. I think a cyber assistant could have drastic impacts on
the way people view the security of their system, bringing an important topic
out from under the hood and into the hands of those who it matters most to.

