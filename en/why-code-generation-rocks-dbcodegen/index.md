# The Joy of Code Generation — Marking My First Integrated Code Generator, DBCodeGen


It suddenly struck me that an engine which supports rapid development needs, besides a good foundation and convenient templates, a complete set of development tools. That toolset should include automated code generation — the way Visual Studio generates an MFC application for you. It saves us a ton of unnecessary work.

Nobody who has used a code generator needs convincing. Back when I was building each system, there was always some repetitive work, and that work was awkward to handle with templates or interface base classes — every time it was copy, paste, then modify, which was a real pain. Then one day a colleague said to me, you can write a program to generate that code. And that's how, after eating rice for 30 years, I finally learned you can make congee.

After that the process wasn't anything novel: whatever code could be generated automatically, got generated automatically. Define an object's fields, and out comes the SQL to create the database table, the matching server-side class for reading and storing that data object plus its operation code, the message classes for talking to the client, the client-side message definitions — and of course that wasn't the end of it. In the future I'd add the client-side handling code and so on. Anything that could be generated automatically, any code that basically never changes, all of it gets generated. After that I found that writing a program no longer meant getting bogged down in all this fiddly work; I freed up more time and energy for the things a system actually deserves some brainpower spent on.

Looking back, automatic code generation has been around since the day I started coding — the simplest MAIN function in TC, the APP wizard in VC6, they all generate some code for you to save you keystrokes. I feel pretty dumb now, and pretty unobservant: something that was happening right next to me all along, and I never made use of it. Maybe that's the difference between the exceptional and the ordinary.

