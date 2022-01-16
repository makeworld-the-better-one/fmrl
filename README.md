# fmrl

I find Discord status messages fun, and a nice way to make someone laugh, see how friends or doing, or find out about new games and music. But what if we freed them from Discord? What could they become?

And for the oldies in the crowd: remember AIM away messages?

Enter fmrl. Pronounced like the word ephemeral, it is a decentralized protocol to read user statuses and set your own. From "Chilling" to "Get me out of here" to "This is the best album ever", fmrl is there for you.

## Current Status

As of January 2022:

I am developing the reference server implementation, and updating the spec as I encounter issues. The spec will be versioned after that is done. A few other developers are working on their own server and client implementations. These will be linked here once the spec is versioned.

## Branding

Please style the name as fmrl or if necessary, FMRL. Never Fmrl.

## Philosophy

- No feeds, just one message from people you follow
- No history, just the current message
- No paragraphs, just a statement

## Software

Coming soon! If you implement any software let me know. Please make it clear in your project README what version of the spec your software supports.

## Specification

Please see the [Specification](./spec.md).

## FAQ

**Q: Are you hoping to change the world?**

No. I'm hoping to create something fun, that some people would use and get a kick out of.

**Q: What's your target audience?**

I'd love to see non-techy people using this as way to keep up with and check in on friends, like how Discord statuses are used by some now. fmrl could be used by those who aren't using Discord already, or by those who see it as an improvement for what Discord does.

To do this requires good web interfaces exist, as well as a good community of servers ready to host people.

Aside from that audience, maybe this is something that the [Mastodon](https://joinmastodon.org/) crowd will use, people who are already willing to try something new and understand the merits of decentralization. Some sort of integration like a Mastdon+fmrl client would be really cool.

**Q: Isn't this just like [finger](https://en.wikipedia.org/wiki/Finger_%28protocol%29)?**

The main difference is that the data is structured, while finger data is just a text file. Structured data allows for more flexibility in displaying statuses. It also allows for non-text, like the avatars fmrl offers.

**Q: Why not build off [Webfinger](https://www.packetizer.com/ws/webfinger/)?**

Webfinger is also complex, extensible, and ugly. It wasn't designed with something like fmrl in mind. It also provides no standard way for clients update data. There would be little advantage in using it compared to making something new that is simpler.

## Discussion

To propose updates to the spec, please use GitHub issues. For general discussion and questions, join us on IRC at `#fmrl` on libera.chat. Or `#fmrl:libera.chat` for [Matrix](https://matrix.org/) users.
